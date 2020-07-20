---
layout: post
title:  "Getting most recent GKE security patch version available, with gcloud and yq 1.6"
date:   2020-07-20 09:27:00 -0700
categories: bash gcp yq
---

i don't know _why_ GCP makes this so frickin hard, but it does. querying their container API for available GKE security patch versions works pretty well at the surface if you are a human accustomed to clickops - 

{% highlight bash %}
$ gcloud --project my-fancy-project container get-server-config --zone us-west1-b
# this gives me a ton of contextual info, which i kind of like
#     when i'm first starting out or exploring.
Fetching server config for us-west1-b
channels:
- channel: RAPID
  defaultVersion: 1.17.6-gke.11
  validVersions:
  - 1.17.7-gke.15
  - 1.17.6-gke.11
- channel: REGULAR
  defaultVersion: 1.16.9-gke.6
  validVersions:
  - 1.16.11-gke.5
  - 1.16.9-gke.6
- channel: STABLE
  defaultVersion: 1.14.10-gke.36
  validVersions:
  - 1.15.12-gke.2
  - 1.14.10-gke.46
  - 1.14.10-gke.36
defaultClusterVersion: 1.14.10-gke.36
defaultImageType: COS
validImageTypes:
- UBUNTU
- COS_CONTAINERD
- UBUNTU_CONTAINERD
- WINDOWS_SAC
- WINDOWS_LTSC
- COS
# and it also gives me information about Master and Node versions
validMasterVersions:
- 1.16.11-gke.5
- 1.16.10-gke.8
# ...and quite a few other master versions that i don't want
validNodeVersions:
- 1.16.11-gke.5
- 1.16.10-gke.8
# ...and about a gazillion node versions for minor versions
#     i don't care about at all
{% endhighlight %}


okay, so this is all helpful if i'm a human. but if i'm a robot, i probably don't care about all the context like what's going on in the various channels. let's filter it down to only the NodeVersions:

{% highlight bash %}

$ gcloud --project my-fancy-project container get-server-config --zone us-west1-b --format yaml | yq r --printMode v - 'validNodeVersions'

{% endhighlight %}


furthermore, since i'm using this code in the context of automatically installing patches for my current minor version, i don't care one whit about anything other than the minor Kubernetes version i'm working on, which happens to be 1.15 at the time of this writing. how do i filter out all the extras and just get the master and node versions for the minor.patch versions i'm working on?

{% highlight bash %}

$ gcloud --project my-fancy-project container get-server-config --zone us-west1-b --format yaml | yq r --printMode v - 'validNodeVersions.(.==1.15.12*)'
Fetching server config for us-west1-b
1.15.12-gke.9
1.15.12-gke.6
1.15.12-gke.3
1.15.12-gke.2
{% endhighlight %}


well, that's better, but it still returns a list of things  i only want the most _recent_ patch version for 1.15.12. so i pipe it to a `head`.

{% highlight bash %}

$ gcloud --project my-fancy-project container get-server-config --zone us-west1-b --format yaml | yq r --printMode v - 'validNodeVersions.(.==1.15.12*)' | head -1
Fetching server config for us-west1-b
1.15.12-gke.9

{% endhighlight %}

now, it still outputs little preamble about what zone it's working in, but that's fine for now.

for the record, i futzed with this for a couple of hours, trying to use `jq`. i just couldn't get `jq` to parse the "list" that was returned. so that's why i ended up going with `yq` on this.