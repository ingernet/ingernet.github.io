---
layout: page
title: "Creating and Using Service Accounts and Custom Roles in GCP"
date:   2020-08-14 11:50:00 -0700
categories: bash gcp iam 
---


**Disclaimer:** this is not comprehensive, more of a scratch pad for me to make notes on my process of:
- creating a custom role
- creating a service account
- assigning the new custom role to the new service account

The code snippets here are hardcoded versions of snippets from a script I wrote that creates or updates custom roles, and keeps the spec files updated with the new etags as the role is updated. It's just faster here for me to leave all that logic out of this doc. If you want help writing your own script for this, drop me a line.

Also, you can create roles at the org level (if you have high enough permissions), but for this doc we're just doing it at the project level.

*Requirements:*
- gcloud executable
- yq
- IAM admin permissions within your project

## Create a custom role
**Define a custom role in a yaml spec file:**
```
$ cat mynewrole.yaml
title: EngDeploybotTest
etag:
description: "Service Account to run deployments"
stage: "ALPHA"
includedPermissions:
- container.clusters.get
- container.clusters.getCredentials
- container.configMaps.create
- deploymentmanager.deployments.get
```

**Create the custom role in your project:**
```
$ role_id=$(yq r mynewrole.yaml title)
$ gcloud iam roles create ${role_id} --project=my-fancy-project --file=mynewrole.yaml
```


## Create a service account ##
```
$ gcloud --project=my-fancy-project iam service-accounts create \
my-svc-acct@my-fancy-project.iam.gserviceaccount.com \
--description="Service Account used for deploying things" \
--display-name="my-svc-acct"
```

## Generate and store its keys
Create and store the key that you'll need to activate this account. GCP recommends using JSON format intead of P12. In this example I'm just gonna save these locally; additionally, you may want to save this JSON file in a secrets store like Google Secrets Manager or LastPass, in case you need to give somebody else access to use this service account.

Anyway, here's me generating the service account keys. Of note here is that I'm using the $HOME variable to point to my homedir, instead of my usual "~" shorthand. This is because later on in the service account activation command, the _key-file_ attribute isn't able to expand the tilde, but it does recognize the $HOME environment variable.
```
$gcloud --project=my-fancy-project iam service-accounts keys create \
$HOME/.config/gcloud/gcp-service-acct-keys/my-svc-acct.json \
--iam-account my-svc-acct@my-fancy-project.iam.gserviceaccount.com
```

## Give the service account your new custom role ##
```
$ gcloud projects add-iam-policy-binding my-fancy-project \
--member=serviceAccount:my-svc-acct@my-fancy-project.iam.gserviceaccount.com --role=roles/EngDeploybotTest
```


## Activate the service account ##

```
$ gcloud --project=my-fancy-project auth activate-service-account \
my-svc-acct@my-fancy-project.iam.gserviceaccount.com \
--key-file=$HOME/.config/gcloud/gcp-service-acct-keys/my-svc-acct.json
```

Activating this account for the very first time will automatically set it as the active account in your auth list. You can confirm this: 
```
$ gcloud auth list
                      Credentialed Accounts
ACTIVE  ACCOUNT
*       my-svc-acct@my-fancy-project.iam.gserviceaccount.com
        inger.klekacz@dayjob.com
        ingernet@personal-account.com
        another-svc-acct@my-other-project.iam.gserviceaccount.com

To set the active account, run:
    $ gcloud config set account `ACCOUNT`

```

## Run Something as your service account ##
Once you've activated the service account, everything you do, you're doing as that account - so if you're accustomed to running as Project Owner, just get ready for the occasional odd IAM error.

```
# don't have this permission as this service account, so it won't work
$ gcloud --project=my-fancy-project deployment-manager deployments list
ERROR: (gcloud.deployment-manager.deployments.list) ResponseError: code=403, message=Required 'deploymentmanager.deployments.list' permission for 'projects/my-fancy-project'

# but this one should
$ gcloud --project=my-fancy-project deployment-manager deployments describe my-fancy-deployment
---
fingerprint: YI7lnJL964cKTMGWIMfYEg==
id: '5625931904623267331'
insertTime: '2020-08-14T13:15:40.454-07:00'
labels:
- key: owner
  value: deployingCoworkerName
- key: project
  value: fanciness
- key: gcp_env
  value: sandbox
manifest: manifest-1597436566734
name: my-fancy-cluster-deployment
operation:
  endTime: '2020-08-14T13:25:51.102-07:00'
  name: operation-1597436566660-5acdc2f1b6461-9c0dd1dc-83cd6fdf
  operationType: update
  progress: 100
  startTime: '2020-08-14T13:22:46.767-07:00'
  status: DONE
  user: deploying-coworker@my-fancy-project.iam.gserviceaccount.com
NAME                  TYPE                                   STATE      INTENT
my-fancy-cluster       container.v1.cluster                   COMPLETED
my-fancy-cluster-type  deploymentmanager.v2beta.typeProvider  COMPLETED

```

## Related Links ##
- [GCP: Managing Service Accounts](https://cloud.google.com/iam/docs/creating-managing-service-accounts)
- [GCP: Managing Service Account Keys](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)
- [GCP: Activating a Service Account](https://cloud.google.com/sdk/gcloud/reference/auth/activate-service-account)
- [GCP: Assigning a Project Role to a Service Account](https://cloud.google.com/iam/docs/granting-changing-revoking-access#updating-gcloud)
- [GCP: IAM Member Types](https://cloud.google.com/iam/docs/reference/rest/v1/Policy#Binding)