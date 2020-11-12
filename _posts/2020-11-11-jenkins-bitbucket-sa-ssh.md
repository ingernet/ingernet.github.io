---
layout: default
title: "Access Bitbucket repos from Jenkins with a GCP Service Account"
date:   2020-11-11 17:20:00 -0700
categories: gcp iam sa
---

This process always feels a little noodly to me, so I thought I'd write it down just in case.

## TL;DR ##

1. create a service account (SA) in GCP
2. generate an SSH key for that service account's use and save both public and private files to some central secrets store - LastPass, Google Secrets, whatever
3. add the public half to Bitbucket > [repo name] > Settings > Access keys > Add key
4. add the private half to Jenkins > Manage Jenkins > Manage Credentials > Jenkins (global) > Add Credentials
5. set up a test pipeline that basically only clones the repo.
6. (details below, after the story about my trip to Italy)

## Backstory ##

Lemme tell you about the time I inherited a wizened Jenkins instance and had to hook it up to Bitbucket with a new service account. 

The company I was at was migrating a bunch of assets from a few different places into a central "repository" GCP project, we'll call it `ingernet-central` - my Jenkins server, some buckets, etc. In the interest of best practices, I was also creating at least one service account in this `ingernet-central` project to handle a variety of tasks, among them cloning a Bitbucket repo. This account replaced the service account that lived in one of the soon-to-be-decommissioned projects, so it was important that I nail the permissions both in IAM and in the `git` sphere.


## The problem ##

How do I set up a service account (SA) and give it access to my private repo? The person who did this work before me basically muttered, "Phew, so THAT happened," and wandered off into the desert to go find his center again.

Now, y'all might be making fun of me for not being super clear on how to enable a service account with Bitbucket perms, but can I  just say? I had just finished upgrading the Jenkins instance itself (we were 14 minor versions behind), deleting 24 unused plugins, doing security updates on 20 more, and reconfiguring pipelines affected by breaking changes. I basically came out of the dark maw of that cave like every Jenkins admin does:

![Every Jenkins Admin Ever](/assets/jenkins-admin.png)

The thing with Jenkins is that there's a plugin for everything, which is great, but that also means that the eleventy-one plugins you have installed to do things like inject timestamps into console output need near-constant maintenance.

Anyway. You've been warned.

## One way to handle the situation ##

Just to reiterate that subhead, this is only _one way_ to deal with things. Another thing with Jenkins is that there are at least 4 ways to do anything. Which is great when you're setting up a greenfield project, but not so great when you're the third or fifth person to inherit a server that's seen some miles, and the field is _decidedly_ brown.

![There's some lovely filth down here](/assets/jenkins-brownfield.jpg)
<figcaption>Me sifting through vestigial Jenkins plugins: "Ooo, Dennis, there's some lovely filth down here"</figcaption>

...But this way worked for me.

### 1. Set up the service account ###

First, create the account:
```
$ gcloud iam service-accounts create jenkinsuser \
    --description="Jenkins Service Account" \
    --display-name="ingernet-central Jenkins SA" \
    --project="ingernet-central"
```

You may also need to give it some special IAM permissions for interacting with Cloud Storage buckets or whatever:

```
gcloud projects add-iam-policy-binding ingernet-central \
    --member="serviceAccount:jenkinsuser@ingernet-central.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"
```

### 2. Create its SSH Key ###

This part was pretty straightforward, if you've ever generated an SSH key. You can generate an SSH key in a wide variety of ways, but I was on a Linux box, so I just used `ssh-keygen`. You can see below that I gave it a custom name so that it wouldn't collide with my original `id_rsa` files:

```
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/inger.klekacz/.ssh/id_rsa): /home/inger.klekacz/.ssh/id_rsa_jenkins_ingernet_central_sa     
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/inger.klekacz/.ssh/id_rsa_jenkins_ingernet_central_sa.
Your public key has been saved in /home/inger.klekacz/.ssh/id_rsa_jenkins_ingernet_central_sa.pub.
The key fingerprint is:
SHA256:<redacted> inger.klekacz@baby-groot
The key's randomart image is:
<redacted>
```

**Don't forget!** Add a copy of these files to a secrets store that your teammates can get to, or they will (eventually) curse your name. I stuck mine in a LastPass shared folder. And if you used a passphrase to generate them, make sure to store that passphrase in a way that it's inarguably tied to those files. FutureYou will thank you.

### 3. Add your public key to Bitbucket Repo ###

We were hosted on Bitbucket Cloud, so these instructions apply to that setup. Your mileage may vary. You could do this with the Bitbucket API, but since we do it so infrequently, it didn't seem worth the time to write the code. I just used the GUI:

1. Log in to your Bitbucket server (mine is https://bitbucket.org)
2. Click on *Repositories* in the left nav
3. Search for your repo in the Search field if you can't just find it in the list, and click on it
4. In the repo's left nav, click on *Repository settings*
5. Click *Access keys* in the left nav
6. Click the **Add key** button in the main pane
7. In the **Label** field, give it a useful name. Since I'd named my SSH files after the SA and the project, I observed the naming schema here, too: `jenkins_ingernet_central_sa`
8. Print to screen the contents of your `.pub` file - every OS does it differently, so you're best off just reading [Bitbucket's helpful guide](https://support.atlassian.com/bitbucket-cloud/docs/set-up-an-ssh-key/) on how to do that. Paste the contents of your .`pub` file into the **Key** field.
9. Click on the **Add SSH key** button.

### 4. Add your private key to Jenkins' Credentials Store ###

The Jenkins instance I'm looking at for reference is in the 2.249 family, and using v2.23.X of Credentials; if you're using different versions, your experience may vary somewhat from these instructions, but the principle is the same.

1. Log in to your Jenkins website. (duh)
2. Navigate to: Jenkins > Manage Jenkins > Manage Credentials > Jenkins (global) > Add Credentials
3. In the **Kind** drop down menu, choose "SSH Username with private key"
4. Scope: keep this Global
5. ID: Leave the ID blank, it will be autogenerated.
6. Description: `jenkins_ingernet_central_sa Bitbucket SSH Access Key`
7. Username: leave blank
8. Private key: click the **Enter directly** radio button, then paste the contents of your *private* key into the **Key** field. 

**Don't forget!** line formatting is very important here. Follow the instructions on that Bitbucket doc to the letter. Make sure you have a blank line after your last line of key contents.
9. Passphrase: if you entered a passphrase while generating this SSH key, enter that here.
10. Save your changes.

### 5. Set up a test pipeline ###

This pipeline's only job is to clone the repo to Jenkins and prove to you that it can see its local clone.

1. Navigate back up to the top level of your Jenkins website.
2. In the left nav, click **New Item**
3. Call it "Test-ServiceAccount-Bitbucket" and choose "Pipeline" for its type.
4. Put something useful in the **Description** field for the next person who has to cleanup old pipelines.
5. In the *Pipeline* section:
    - Definition: choose "Pipeline script from SCM"
    - SCM: choose "Git"
    - Magically appearing *Repositories* subsection:
        - Repository URL: the URL you would use to clone the repo <u>with SSH</u>, e.g. ssh://bitbucket.org/my-organization/my-repo-name.git
        - Credentials: this is where your hard work pays off! Choose the key you just created - it'll show up as the string you provided in step 6 of the last section.
6. FOR THE LOVE OF GOD, THE BUILD SECTION IS MISSING AFTER THE UPGRADE


# (┛ಠ_ಠ)┛彡┻━┻ #

I am going to go ahead and commit all of these repo changes and put this article away for the night and not at all be furious at fucking jenkins



## Related Links ##
- [GCP: Creating Service Accounts](https://cloud.google.com/iam/docs/creating-managing-service-accounts#iam-service-accounts-create-gcloud)
- [Bitbucket: Set up an SSH Key](https://support.atlassian.com/bitbucket-cloud/docs/set-up-an-ssh-key/)