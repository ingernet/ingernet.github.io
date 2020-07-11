---
layout: post
title:  "Running GCP cloud_sql_proxy as a Service"
date:   2020-06-23 13:27:00 -0700
categories: gcp bash
---

setting cloud_sql_proxy up to run in the background for automated db work is deceptively simple...

but getting it right is a little noodly. the trick is to let the cloud_sql_proxy get settled before trying to interact with it. 

this script is a utility that's intended to be used with some adaptation by database maintenance scripts, but could also be used by humans. 

```
# i recommend that you use Google Secret Manager or whatever password CLI tool you have to set these variables
# to the output of a secrets manager query to keep that ish out of a repo.

db_connect_string=$(gcloud --project=my-gcp-project secrets versions access latest --secret="db-connect-string")
db_user=$(gcloud --project=my-gcp-project secrets versions access latest --secret="db-user" | jq -r '.username')
db_pass=$(gcloud --project=my-gcp-project secrets versions access latest --secret="db-user" | jq -r '.password')


# connect to the cloud sql proxy and run it in the background, then
# output its jibber jabber to a log and run it in the background with &
# in the line below, i set the port to 3308 to avoid mashing an existing port 3306 connection you might have.

cloud_sql_proxy -instances=${db_connect_string}=tcp:3308 > cloud_sql.log 2>&1 &

# the secret sauce: WAIT for it to be ready before you try to connect, bro

sleep 5

# now...pounce

mysql -u ${db_user} -p${db_pass} --host 127.0.0.1 --port 3308 -e "show databases;"

# end the connection now that you're done with it
# the -p option in xargs allows for a confirmation from you. remove that argument if you trust the robots; i typically don't.

ps -ef | grep cloud_sql_proxy | head -1 | awk '{print $2}' | xargs -p kill
```