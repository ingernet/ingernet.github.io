---
layout: post
title:  "Expanding bash variables inside jq"
date:   2020-04-27 20:00:00 -0700
categories: bash jq
---
# Expanding variables inside jq

It took me 3.5 hours to figure out the hard way that JQ supports variables in a simple but peculiar way. 

I hope it really helps somebody else.

## How to do it

Here's what the data set looks like, btw:
{% highlight bash %}

$ gcloud auth list --format=json
[
  {
    "account": "inger.klekacz@workaccount.com",
    "status": ""
  },
  {
    "account": "ingernet@personalaccount.com",
    "status": "ACTIVE"
  }
]

{% endhighlight %}

Now, remember, bash doesn't expand variables inside single quotes:
{% highlight bash %}
$ tmp_authed_user=$(gcloud auth list --format=json | jq -r '.[] | select(.status=="ACTIVE") | .account')

$ echo '$tmp_authed_user'
$tmp_authed_user            # boo hiss

$ echo "$tmp_authed_user"
ingernet@personalaccount.com 
{% endhighlight %}

Hurray but still not enough, yet, because we _still_ have to somehow jam it between two single quotes like `jq` likes.

{% highlight bash %}
$ jqstring="'.bindings[] | select(.members[] | contains("'"'${tmp_authed_user}'"'")) | .role'"     # kill me now

{% endhighlight %}

Let's see if it runs, I guess?

{% highlight bash %}
$ echo $jqstring
'.bindings[] | select(.members[] | contains("ingernet@personalaccount.com")) | .role'  

{% endhighlight %}

I mean, I'll try it? Hold my beer

{% highlight bash %}
$ gcloud projects get-iam-policy my-fancy-project --format=json | jq $jqstring
jq: error: syntax error, unexpected INVALID_CHARACTER, expecting $end (Unix shell quoting issues?) at <top-level>, line 1:
'.bindings[]                 
{% endhighlight %}

_wheeeeeeee_

Well, dear reader, listen close: I was definitely overthinking things. Here's how you do it:

{% highlight bash %}
# unset that ungodly jqstring, ain't nobody got time for that
$ unset jqstring

# set the user variable
$ tmp_authed_user=$(gcloud auth list --format=json | jq -r '.[] | select(.status=="ACTIVE") | .account')

# pass it into the jq command as an argument, making sure to wrap it in quotes first - 
# notice that i named the arg "authed_user" to show that it's actually getting used 
# inside the scope of jq instead of just being the bash variable

$ gcloud projects get-iam-policy my-fancy-project --format=json | jq --arg authed_user "${tmp_authed_user}" -r '.bindings[] | select(.members[] | contains($authed_user)) | .role'
roles/owner
roles/resourcemanager.projectMover
roles/secretmanager.admin
{% endhighlight %}

Easy peasy. Right? Uggh.

## Why would you do that?

The most immediate application that comes to mind is if you're looping through a list of users and want to get the IAM credentials for all of them (since that's what I was trying to do when I went down this rabbit hole). But I imagine this will be helpful for running a jq query against _any_ list of strings - `for x in y`, etc. 

## Caveats

The code above doesn't grab ALL of the credentials - only the IAM creds assigned at the GCP project level. As of right now, the gcloud SDK doesn't pull in creds inherited from folders, which is a drag. But this gist isn't about that. Anyway, just a heads up.