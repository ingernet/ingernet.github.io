---
layout: page
title: "GCP: What Nobody Tells You About Attaching an Existing Persistent Disk to a Compute VM"
date:   2020-10-02 08:20:00 -0700
categories: gcp compute
---


(This probably isn't the _only_ thing that nobody tells you, but this is the one that I learned about.)

TODAY I LEARNED that when you attach a persistent disk to more than one VM, that disk reverts to read-only mode on ALL VMs. That's right! While it is technically a network mount, a persistent disk is not the same as an NFS mount in traditional systems networking. The phenomenon of sharing persistent disks between instances is [documented](https://cloud.google.com/compute/docs/disks/add-persistent-disk#use_multi_instances), but somehow I missed it until I'd been working the issue for a day and a half, and it certainly doesn't address the issue of what happens when you have an instance already attached as a read-write.

## Why this is tricky ##

There does't appear to be any kind of safeguard or warning to let you know, when you're starting to attach the disk to a new VM, that it will impact existing disk attachments to other VMs.

**To more explicitly paint the picture:**

- You have Persistent Disk pd-001 mounted R/W to VM vm-001.
- For some reason, you (or a coworker) decide that you need to access the contents of pd-001 from a second VM, vm-002. So you mount disk pd-001 to vm-002 as a read-only disk. No big deal, everything is fine, you can read it, everyone is happy
- Later that night, a cron kicks off on vm-001 that does a `gsutil rsync` or some other process that attempts to update the contents of pd-001. That cron fails. Your log files fill up. Depending on your infra monitoring, you might get paged. You might exclaim, "But nothing changed!" But you would be wrong.

There doesn't appear to be any way to set up guardrails for this, except for IAM - but that will only prevent unauthorized users from attaching disks. That doesn't prevent YesterYou from ruining the day of Current You.

![Like Dexter, But Make It Engineering](/assets/debugging-sticker.jpg)

Anyway, consider this literally the only cautionary tale on the interwebs about this phenomenon - believe me, I googled and googled. Perhaps the next googler will find this article.


## Related Links ##
- [GCP: Attaching Persistent Disk](https://cloud.google.com/compute/docs/disks/add-persistent-disk#use_multi_instances)
