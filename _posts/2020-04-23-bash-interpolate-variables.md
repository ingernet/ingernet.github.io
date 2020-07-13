---
layout: post
title:  "Interpolating Secrets in bash"
date:   2020-04-23 20:00:00 -0700
categories: bash jq
---
## Interpolating Variables with Bash ##

(With a MAJOR hat-tip to [Stark and Wayne](https://starkandwayne.com/blog/bashing-your-yaml/), whose only downfall was that their CMS isn't optimized for copypasta for us devs.)

Okay so the thing here is, you need a sample/template file that has the values you're expecting to replace:

template.yaml:
```
---
# ${sample}
db:
  password: "${db_password}"
  username: '${db_username}'
```

Now, you'll want a bash script that jams your secrets into the variables referenced by your template file. I'm calling it `interpolate_secrets.sh` for clarity's sake:

```
#!/bin/bash

export sample="Hello Yaml!"
export db_password="mySecretDbPassword1234"
export db_username="dbuser"

rm -f final.yaml temp.yaml  
( echo "cat <<EOF >final.yaml";
  cat template.yaml;
  printf "\nEOF";
) >temp.yaml
. temp.yaml
```

Make your script executable:
```
$ chmod +x interpolate_secrets.sh
```

...and then run it:
```
$ ./interpolate_secrets.sh
```

...and then take a look at your final interpolated results in final.yml:
```
$ cat final.yaml

```

...don't forget to delete your `temp.yaml` and `final.yaml` files when you're done, otherwise you run the risk of checking those files into your repo! You can edit the script to remove `temp.yaml` after interpolation is done. Or add both of those files to your .gitignore.

## Security Note ##
So, this is a fine MVP - but here's where it's really going to come in handy: with pulling down secrets from whatever keystore you're using.

It's a not-great idea to export your secrets to environment variables. It's less bad here because you're running the `interpolate_secrets.sh` file, and not `source`ing it (which would save these secrets to your actual environment variables for Mr. Robot to snatch). But because you're dictating the actual contents of your secrets in this code, you couldn't check this code into a repo because then you would be That Guy - The Guy Who Checked In Secrets.

Don't be That Guy.

I'm just telling you this because many of us seniors out there have been That Guy, and you totally don't have to make that mistake; we've already made it for you. Use this code as a starting point, but instead of setting secrets to strings inline, feel free to use command line tools to pull secrets from central datastores:

in GCP:
```
export mybigsecret=$(gcloud secrets versions access latest --secret="my-secret")
```
or in AWS:
```
export mybigsecret=$(aws secretsmanager get-secret-value --secret-id my-secret | jq '.SecretString')
```
or even in CredHub:
```
export mybigsecret=$(credhub get -n '/my-secret' -q)
```
...you can also use the [LastPass CLI](https://github.com/lastpass/lastpass-cli) to grab centrally stored secrets, but that's a whole other gist that I don't feel like writing.