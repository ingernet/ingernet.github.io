---
layout: post
title:  "Querying GCP Firestore with Python3"
date:   2020-06-07 13:00:00 -0700
categories: gcp python3
---

*Note: this assumes that you've set up your credentials and whatnot.*

Collection name: collection1

Schema contents:
{% highlight python %}

  {'lname': 'Pasha', 'fname': 'Sasha'}
  {'fname': 'Rasha', 'lname': 'Pasha'}
  {'lname': 'Pasha', 'fname': 'Natasha', 'faves': '{"animal":"giraffe","color":"aqua","flavor":"mint"}'}
{% endhighlight %}


---
## First, import the libs you need and define your collection:
{% highlight python %}

from google.cloud import firestore
import json

db = firestore.Client()
col_ref = db.collection(u'collection1')

{% endhighlight %}


---

## List all things in collection1:
{% highlight python %}

# list everything in collection1
for x in col_ref.stream():
    y = x.to_dict()
    print(u'{} => {}'.format(x.id, y))
    if "faves" in y:
        print("faves:")
        z = json.loads(y["faves"])
        for fave in z:
            print("{}: {}".format(fave, z[fave]))
{% endhighlight %}


## Find only the people with last name of Pasha
{% highlight python %}

allpashas = col_ref.where(u'lname', u'==', u'Pasha').stream()
for ap in allpashas:
    p = ap.to_dict()
    print(p["fname"], p["lname"])
{% endhighlight %}


## Find only Natasha Pasha and display her name AND favorites:
{% highlight python %}

natnat = col_ref.where(u'lname', u'==', u'Pasha').where(u'fname', u'==', u'Natasha').stream()
for n in natnat:
    np = n.to_dict()
    print(np["fname"],np["lname"])
    if "faves" in np:
        print("  {}'s faves:".format(np["fname"]))
        ff = json.loads(np["faves"])
        for f in ff:
            print("  - {}: {}".format(f, ff[f]))
{% endhighlight %}
