---
layout: page
title: "How to Avoid `gcloud` Race Conditions in your CI/CD"
date:   2020-12-03 17:20:00 -0700
categories: gcp k8s cicd
---

Ran into a fun little snag yesterday. 


## TL;DR ##

1. Using a central kubeconfig file is fine for proof of concept work, but not recommended in production deployment environments; it's much better to have one kubeconfig file per deployment pipeline
2. kubeconfig files can be stored in version control
3. `gcloud container clusters get-credentials` changes the context in which kubectl runs
4. If pipeline A changes the context while pipeline B is running, you run the risk of pipeline B failing (bad) or accidentally running against a different context, i.e., the cluster used by pipeline A (bad, very bad, quite terrifying actually).

## The scene ##
So, we have this Jenkins job that builds a Java app. Upon its success, that app is immediately deployed (via two different other jobs) to two different GKE workloads in the same cluster.

These workload deployment jobs have always coexisted. It has NEVER been a problem to have them run in tandem. Until yesterday.

## What broke? ##

It's so silly, really. We've just been getting lucky this whole time. Our Jenkins box is somewhat undersized when it comes to giving the devs what they want when they want it, but it's hard to justify beefing it up, because we're not a giant team of devs building 100 jobs an hour, so we're kind of limited to 2 simultaneous Jenkins executors. It works fine. Everything is fine. It's fine.

<p class="centered"><img src="/assets/greatgoodokaynotokayihateyoufine.gif" alt="The scale goes: great, good, okay, not okay, I hate you, fine."></p>
<figcaption class="centered">Ask me how I feel about Jenkins</figcaption>

But then you get these obnoxious pipelines that have to remain active Because Reasons™, but they CERTAINLY don't need to poll repos that haven't changed in months...every 5 minutes. Because every time one of those six (6!) pipelines polls the repo, it takes up a spot in the Jenkins build queue.

So Girl Genius over here (points to self) decided to reduce the polling frequency of those pipelines.

What I didn't know was that those annoying little polling jobs were actually providing the necessary buffer between the two GKE deployment jobs, keeping them from editing the same kubeconfig file at the same time.

So yesterday, out of the blue, we started getting deployment failures from Jenkins, barking about how it couldn't process YAML and how `kubectl` didn't have a resource called `deployments`. (Excuse me? it most certainly does!)

Turns out, both of the GKE workload jobs were running a `gcloud` command to ensure that the Jenkins service account actually had the credentials required to even update the cluster:

```
gcloud container clusters get-credentials my-fancy-cluster
```

That command does two things that we care about: 
1. it updates the central kubeconfig file, located in - in Jenkins' case - /var/lib/jenkins/.kube/config, with an access token
2. it sets the "current context" to the cluster from which you received the credentials, ensuring that when you run something like `kubectl get deployments`, it gives you the deployments for `my-fancy-cluster` and not `this-other-cluster-over-here`

This is fine, having a central kubeconfig file, unless you happen to have two pipelines running that get-credentials command at the same time, in case one of two bad things can happen, the second of which actually occurred to me while I was writing this blog post:

- bad: that centralized kubeconfig file gets munged and at least one of the jobs fails because it can't parse the YAML
- REALLY bad: two completely unrelated jobs pull credentials for _different clusters_, and whichever of those jobs runs last ends up doing unexpected things to the _wrong cluster_

<p class="centered"><img src="https://media.giphy.com/media/MeFm94nKdkAOA/giphy.gif" alt="Ross from 'Friends' anxiously covering his eyes"></p>
<figcaption class="centered">oh no</figcaption>

## Oh God. How Do I...NOT Do That? ##

I reached out to [DoiT International](https://www.doit-intl.com/) - our GCP support team - for an assist, and they quickly came back with a great solution: give the builds their own mini kubeconfig files!

Mini kubeconfig files! Awwww! Adorable. Anyway, here's how that works, _many_ thanks to DoiT and [Ahmet Alp Balkan](https://ahmet.im/blog/authenticating-to-gke-without-gcloud/) of Google.

(A lot of this stuff is covered in more detail by Ahmet in the link above, so I'm going to gloss over a bunch of details here. But I observe some server security in my flow that isn't covered in his blog post.)

## Part 1: the Googly and Bashy bits ##
1. In GCP, create a service account for use in deployment
2. Create a service account key for that account. Download it locally.
3. Add that service account key to your Google Secrets Manager:
```
gcloud --project=my-fancy-project secrets create jenkins-sa-key-json --replication-policy="automatic" --data-file=/Users/inger.klekacz/.config/gcloud/svcacct_keys/my-fancy-project-jenkins.json
Created version [1] of the secret [jenkins-sa-key-json].
```
1. While logged in as yourself, an authenticated user with the proper k8s credentials, create a kubeconfig file called _kubeconfig-[cluster-name].yaml_ for the cluster that you want to access (notice that in this example, the cluster name is _my-fancy-cluster_, the zone is _us-west1-b_, and the project name is _my-fancy-project_ - just useful to keep in mind when trying to figure out how the names are generated):
```
# this is bash
GET_CMD="gcloud --project=my-fancy-project container clusters describe my-fancy-cluster --zone=us-west1-b"
cat > kubeconfig-my-fancy-cluster.yaml <<EOF
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: "$(eval "$GET_CMD --format='value(masterAuth.clusterCaCertificate)'")"
    server: "https://$(eval "$GET_CMD --format='value(endpoint)'")"
  name: cluster-1
contexts:
- context:
    cluster: cluster-1
    user: user-1
  name: gke_my-fancy-project_us-west1-b_my-fancy-cluster
current-context: gke_my-fancy-project_us-west1-b_my-fancy-cluster
kind: Config
users:
- name: user-1
  user:
    auth-provider:
      name: gcp
EOF
```
_**Note #1:** this is slightly different from Ahmet's because I prefer expanded YAML for legibility_<br />
_**Note #2:** I took the trouble of naming my cluster because this naming schema matches the one generated by the get-credentials subcommand. It's just easier all around._

1. Because this kubeconfig file doesn't have any secrets in it, you can stick it in your repo with the rest of your deploy code. Hurray!
2. Upload your code to the repo. For the purposes below, we're assuming the file has made it to your master branch somehow.

## Part 2: The CI/CD bits ##

This is written about Jenkins because that's what I'm using. But it's probable that Circle CI, or Travis, or Bitbucket Pipelines, or the team of monkeys you have manually clicking buttons in the hall closet, all run similarly.
1. Generate an SSH keypair on Bitbucket for your service account file to use. Save the keypair and its passphrase somewhere that is accessible to not just you, but to your team. (I know, I harp on this in nearly every post, but the hair-raising tales I could tell about lost keys....)
2. Store the public key on Bitbucket, or Github, wherever your repo is. I'm using Bitbucket.
3. Register the private key in your CI/CD server. In Jenkins, it's Jenkins > Manage Jenkins > Credentials > global scope > Add Credentials (left nav):
    - Kind: SSH Username with Private Key
    - Scope: Global
    - ID: Leave this blank and it will be autogenerated
    - Description: Make this meaningful: "Jenkins SSH Key for Bitbucket" or something like that
    - Username: jenkins
    - Private key: "Enter directly," then paste contents of private key into the Key field
    - Passphrase: If you chose one when setting up this keypair, enter it here.
4. Create a new freestyle project - you can eventually roll this into a Jenkinsfile, but for now let's just have a freestyle project that executes a shell script as its Build step.
5. Set the Source Code Management section to Git, then add the SSH URL of your repo. Click the Credentials drop-down menu and find your recently added SSH key.
6. Leave all the other stuff at defaults except for Build, where you click the "Add build step" drop-down menu and choose _Execute shell_.
7. In the _Command_ field, add this:

```
SA_KEY="temp-sa-key.json"
gcloud --project=leanpath-eng-resources secrets versions access latest --secret jenkins-sa-key-json > ${SA_KEY}
export GOOGLE_APPLICATION_CREDENTIALS="${SA_KEY}"
export KUBECONFIG=kubeconfig-my-fancy-cluster.yaml

echo "-----------------------"
echo ""
kubectl get deployments

rm ${SA_KEY}
```

Saving and running that job:
1. Grabs the SA key that you stored in Google Secrets Manager and stores it in a temporary file
2. Exports a couple of environment variables that are available _only to this job_ - THIS is what saves you from the race condition!
3. Uses that SA key/kubeconfig file combination to auth into the cluster, then generate a list of deployments on the cluster in question. 
4. That last line will _also_ remove the service account key from the artifacts of your build, which is what you want. Never store secrets in artifacts!

If you duplicate the process for a second pipeline using a kubeconfig file for a different cluster, you'll see that the jobs are, indeed, completely isolated from each other, and your worries about accidentally deploying to the wrong cluster are over.

<p class="centered"><img src="https://media.giphy.com/media/vTeqJhmcl98uk/giphy.gif" alt="person very nearly getting dragged up an escalator"></p>
<figcaption class="centered">phew</figcaption>

## Related Links ##
- [GCP: Creating Service Accounts](https://cloud.google.com/iam/docs/creating-managing-service-accounts#iam-service-accounts-create-gcloud)
- [GCP: Creating Service Account Keys](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)
- [Bitbucket: Set up an SSH Key](https://support.atlassian.com/bitbucket-cloud/docs/set-up-an-ssh-key/)
- [GCP: Secret Manager](https://cloud.google.com/secret-manager/docs/creating-and-accessing-secrets)
- [Jenkins: Using Credentials](https://www.jenkins.io/doc/book/using/using-credentials/)
- [Jenkins: Creating a Freestyle Project](https://www.guru99.com/create-builds-jenkins-freestyle-project.html)