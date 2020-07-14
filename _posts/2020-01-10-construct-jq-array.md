---
layout: page
title: "Using JQ to Create a JSON array"
date:   2020-04-15 20:00:00 -0700
categories: bash jq json
---

Ran into this while trying to get the IP address of the *first* VM that belonged to the `credhub` service 
that was installed on my PAS instance in Pivotal Cloud foundry. It took me a bit to 
figure out...even Stack Overflow let me down!
Maybe somebody else will stumble upon it and find it useful.

Given this JSON input:

```
{
  "Tables": [
    {
      "Content": "vms",
      "Header": {
        "active": "Active",
        "az": "AZ",
        "instance": "Instance",
        "ips": "IPs",
        "process_state": "Process State",
        "vm_cid": "VM CID",
        "vm_type": "VM Type"
      },
      "Rows": [
        {
          "active": "true",
          "az": "us-west-2a",
          "instance": "backup_restore/<redacted>",
          "ips": "10.34.4.7",
          "process_state": "running",
          "vm_cid": "<redacted>",
          "vm_type": "t2.micro"
        },
        {
          "active": "true",
          "az": "us-west-2a",
          "instance": "credhub/<redacted>",
          "ips": "10.34.4.13",
          "process_state": "running",
          "vm_cid": "<redacted>",
          "vm_type": "t2.medium"
        },
        {
          "active": "true",
          "az": "us-west-2a",
          "instance": "credhub/<redacted>",
          "ips": "10.34.4.12",
          "process_state": "running",
          "vm_cid": "<redacted>",
          "vm_type": "t2.medium"
        }
      ]
    }
  ]
}
```

I started with just this:

```
$> jq '.Tables[0].Rows[] | select(.instance | startswith("credhub/")) | .ips'
"10.34.4.13"
"10.34.4.12"
```

...and that would have been fine if I had wanted all of the IPs - but I just wanted the *first*.

There was no way to isolate just the first one because the results weren't organized into an array, just belched out by jq. So I messed with map(), etc. (which was totally the wrong direction.)

Today was the day that I learned that you can use JQ to _construct_ things, in addition to _parsing_ things:
https://stedolan.github.io/jq/manual/#TypesandValues:

I basically wrapped everything from `.Tables` to `("credhub/"))` square brackets (`[]`) to construct an array out of the results, then accessed the first item in the array with `[0]`:

```
$> jq '[.Tables[0].Rows[] | select(.instance | startswith("credhub/"))][0] | .ips'
```
