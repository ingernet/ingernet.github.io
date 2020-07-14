---
layout: page
title: "Converting a JSON array to a bash array"
date:   2020-04-15 20:00:00 -0700
categories: bash jq json
---

Bash is the most common language of DevOps because you can guarantee that it's installed on a server. 

Bash is _not_ pretty. It's not an elegant language. It's better than, like, C, but it's still awkward and clunky.

Anyway, I love bash, so that's enough of that.

One of the (great many) things that bash is persnickety about is list format:

- A JSON array looks like this: `[ 1, 'Thing 2', 3 ]`
- A bash array looks like this: `( 1 'Thing 2' 3 )`

### The problem

The problem is twofold:

1. A great many APIs return output in JSON format, but bash has no way to loop through it.
2. Languages that _can_ parse JSON format easily don't always have library support for whatever it is you're interacting with. 
(In my case, I had to revert to bash because GCP is still catching up to AWS in the "API support" race.)

### The solution: bash/sed string manipulation

Below, I run a gcloud command and use [JQ](https://stedolan.github.io/jq) to grab an array from its JSON output, then do a little sed/regex magic and convert it to proper bash format:

```
# using gcloud output as a source because why not use the hardest shit possible

bork=$(gcloud --project=<project-id> container images list-tags us.gcr.io/<project-id>/<image-name>  --filter='tags:DEPLOYED' --format=json | jq '.[0].tags')

echo $bork
[ "260", "61a1d7aef75421f5c209c42304716ba44e86ab7a", "DEPLOYED.2019-11-12T17.04.37.772145800Z", "DEPLOYED.2019-11-13T00.00.29.525908800Z" ]
# ^ output is obviously not a bash array

# strip out all the things you don't want - square brackets and commas
borkstring=$(echo $bork | sed -e 's/\[ //g' -e 's/\ ]//g' -e 's/\,//g')
arr=( $borkstring )
echo $arr
( "260" "61a1d7aef75421f5c209c42304716ba44e86ab7a" "DEPLOYED.2019-11-12T17.04.37.772145800Z" "DEPLOYED.2019-11-13T00.00.29.525908800Z" )
# ^ now THAT is a bash array
```

### Parsing your new bash array
Cool. You've got your bash array. Now what? Loop through it.

Here I'm just using echo inside this loop, but ultimately I used this loop to add tags to a copy of this container image that I stuck in another project. The story behind that decision is long and boring.

```
# loop through that lil bebe
for i in ${arr[@]}; do
    # add a little native bash quote-stripping in there; without it,
    # all your array elements will be wrapped in quotes.
    echo ${i//\"}
done
```

### The Output

```
260
61a1d7aef75421f5c209c42304716ba44e86ab7a
DEPLOYED.2019-11-12T17.04.37.772145800Z
DEPLOYED.2019-11-13T00.00.29.525908800Z
```