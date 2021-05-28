---
layout: page
title: "Back Up Cloud SQL to a Cloud Storage Bucket"
date:   2021-05-27 13:22:00 -0700
categories: gcp bitbucket cicd
---

## TL;DR ##
GCP has a collection of sharp edges in their Cloud SQL offering, most notably surrounding their backup processes. The workaround for this is to use your CI/CD solution to manually back up your Cloud SQL instance's databases to a Cloud Storage bucket.


## What's the point of a backup? ##
In short, disaster recovery. 

The thing with backups is that you don't care about them at all until you really, really need them. So you better have them set up, yo! However, it's not enough to have them set up - you need to confirm that you'll have access to them when you need them.

## Doesn't GCP allow you to back up your databases? ##

Well, yes, BUT. And it's a big but. And like Sir Mix-A-Lot, I also like big butts, but only with two Ts.

<p class="centered"><img src="/assets/gcp-sql-big-butts.gif" width="500px" />
</p><figcaption class="centered">Sir Mix-A-Lot hates big buts</figcaption>

## The big But ##
There are several buts here, sadly none of them with two Ts:

- If your instance is deleted, **its backups go with it.** :exploding_head:
- GCP **doesn't offer deletion prevention** on its SQL instances. :scream:
- GCP doesn't let you access existing backups except to delete them or restore from them. There is no way to manually download the backup files made, or transfer them to a bucket. :unamused:

Each one of these Buts has at least one Issue Tracker ticket associated with it. *The average age of these scary issues in the tracker is a little over 18 months.*<sup>[<a href="https://issuetracker.google.com/issues/171854148" target="_blank">1</a>][<a href="https://issuetracker.google.com/issues/116218535" target="_blank">2</a>][<a href="https://issuetracker.google.com/issues/141349131" target="_blank">3</a>]</sup> (Feel free to go upvote ("star") them if you feel that these remediations should be addressed.)

There are more Buts coming, but I'll just let these soak in for now.

## But(t) there's a workaround ##
And it's an absolute hack and I'm frankly a little miffed that I had to write it.

In short:
1. Set up a Cloud Storage bucket
2. Create a scheduled CI/CD job to run  `gcloud sql export sql`  to the bucket

### Setting up the Cloud Storage bucket ###
Ideally, for disaster recovery purposes, your bucket should:
- live outside the project where the Cloud SQL server is, in case the project is deleted (remember, you are planning for the worst case scenario here).
- be multi-regional at a minimum, in case the region where your instance lives is hit by a meteor (I sleep really well at night, why do you ask?)
- have a retention policy, which prevents objects from being deleted until after they're a certain age.
- have a lifecycle policy that nukes old backups that are older than your retention policy specifies. No need to pay to store backups that aren't useful anymore. We don't need to end up on "Hoarders."

**IAM Considerations**

If the Cloud SQL instance is in Project A, and the bucket is in Project B:
- create a service account in Project B. in Project A, give it `cloudsql.instances.export` (you may also need `cloudsql.databases.get` and `cloudsql.instances.get`.)
- make a note of the Cloud SQL Instance's own service account in Project A (it shows up in the instance's "Overview" page in GCP Console > SQL > [instance name]). In Project B, give _that_ service account the Role of "Storage Object Creator."
- keep in mind that if you have a read replica for your instance, you can do an export from the replica as easily as you can from the primary, so you won't impact production performance.

## What if the database is giant? ##
Here's one of those other Buts I was telling you about. The `gcloud sql` API - I think it's the API, I believe this problem extends to clients as well - **times out after 10 minutes** (although the operation continues to run). So if you have a giant database, for example the 108GB honker I was trying to export - the pipeline step you set up to export the database will inevitably return a "fail." 

There's a workaround for this, too:
1. In your export step, use the `--async` flag to let the export process run without waiting for it to finish: 
```
gcloud sql export sql <instance name> <bucket URI> --async --database=<database name>
```
2. Create a "babysitter" pipeline job that is scheduled for X number of hours after your export pipeline runs - a time when you _know for certain_ that the export file will be up in the Storage bucket. It's not checking for a success code from the export command; rather, it's checking for the presence of the artifact.

Is this a hack? Absolutely. But anybody who's been in Ops for more than 10 minutes will tell you that the entire internet is built upon ridiculous hacks like this, and the only reason this house of cards continues to stand is through the diligence of engineers:

<p class="centered"><img src="https://imgs.xkcd.com/comics/dependency.png" alt="XKCD's dependency map of the internet"></p>
<figcaption class="centered"><a href="https://xkcd.com/2347/" target="_blank">XKCD nails it</a></figcaption>


## Setting up the Pipeline ##
I'm using Bitbucket Pipelines, but this logic will translate to any other CI/CD that supports time-triggered pipelines.
Here, you can see that I have some variables that I set within my repository settings:
- `SA_KEY` - a JSON string that contains the contents of this service account's key
- `SQL_BACKUP_BUCKET_PROJECT` - the name of the project that contains the bucket
- `TEAMS_NOTIFY_CHANNEL_SUCCESS` - the name of the Teams channel that should receive a success notification
- `TEAMS_NOTIFY_CHANNEL_FAIL` - the name of the Teams channel that should receive a failure notification

```
# bitbucket-pipelines.yml
image: google/cloud-sdk:alpine

pipelines:
  custom:
    'Cloud SQL Backup':
    # I run at midnight and my backup takes 30 minutes to run, normally
      - step:
            name: "Export DB"
            caches:
              - docker
            services:
              - docker
            script:
              - |
                echo "activating service account"
                echo ${SA_KEY} > sa_key.json
              - gcloud auth activate-service-account bitbucketsvcacct@my-bucket-project.iam.gserviceaccount.com --key-file=sa_key.json --project=${SQL_BACKUP_BUCKET_PROJECT}
              - NOW=$(date +"%Y%m%d%H%M%SUTC")
              - gcloud --project=my-sql-project sql export sql my-sql-instance gs://my-bucket-name/db-backup-filename-${NOW}.gz  --database=database-to-backup --async
              - rm sa_key.json

    'Backup Babysitter':
    # I run at 2am, giving Cloud SQL Backup plenty of time to finish before looking for the artifact
      - step:
          name: "Look for last night's DB Backup File"
          caches:
            - docker
          services:
            - docker
          script: 
            - echo "activating service account"
            - echo ${SA_KEY} > sa_key.json
            - gcloud auth activate-service-account bitbucketsvcacct@my-bucket-project.iam.gserviceaccount.com --key-file=sa_key.json --project=${SQL_BACKUP_BUCKET_PROJECT}
            - TODAY=$(date +"%Y%m%d")
            - |
              if [[ $(gsutil ls gs://my-bucket-name/db-backup-filename-${TODAY}*) ]]; then
                NOTIFY_MSG="'Notice from my-bucket-project: **YAY!** Database has been backed up to its bucket.'"
                TEAMS_NOTIFY_CHANNEL=$TEAMS_NOTIFY_CHANNEL_SUCCESS
              else
                NOTIFY_MSG="'Notice from my-bucket-project: **OH NO!** Database was not backed up.\n I searched for:\ngsutil ls gs://my-bucket-name/db-backup-filename-${TODAY}*\nGo look into it.'"
                TEAMS_NOTIFY_CHANNEL=$TEAMS_NOTIFY_CHANNEL_FAIL
              fi

              curl -H "Content-Type:application/json" -d "{'text': $NOTIFY_MSG}" "${TEAMS_NOTIFY_CHANNEL}"
            - rm sa_key.json
```

Then I go into Bitbucket and schedule the custom jobs accordingly, to run the backup job at midnight and the artifact checking job at 2am. If your DB takes less than 10 minutes to run, you won't need that babysitter job. But I did, and you might. So here's the code.


## The Takeaway ##

It _is_ possible for you to take effective disaster recovery steps in GCP, but be very conscious of the fact that you are going to have to own the DR process from soup to nuts, and test and wargame appropriately.

