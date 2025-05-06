---
title: "Creating backups for fly.io Volumes"
date: 2025-04-30
lastmod: 2025-05-06
tags: ["self-host", "fly.io", "backup"]
slug: creating-backups-for-fly-io-volumes
---

I have a few applications running on [fly.io](https://fly.io), and some of them need to
keep data in the file system persistently (more precisely, an SQLite database file and
user-submitted data) so that it is not lost after a redeploy or when the Fly Machine running my application is
restarted.

To achieve that, I use [Fly Volumes](https://fly.io/docs/volumes/overview/) which are local
persistent storage for Fly Machines, mounted in my server just like a regular directory. This setup works fine, 
but I began considering how to back up the data stored there.

[Volume snapshots](https://fly.io/docs/volumes/snapshots/) are created automatically on a daily basis and 
retained for 5 days by default. However, there doesn't seem to be an easy (or well-documented) way to implement
a custom backup policy. I wanted the ability to copy the entire directory's content using tools like `rsync` or upload
it to an S3 bucket on my own schedule.

I explored solutions involving `cron` jobs running inside my Fly Machine, but they became overly complicated. 
These approaches required modifying my `Dockerfile` to install additional applications, and I wasn't sure 
how to manage the schedule effectively, especially since I configured my machines to auto-stop to save resources.

Direct SSH connections requires me to use `flyctl` CLI and it wasn't clear to me how to handle authentication
in this case. After some research, I found that I can use [access tokens](https://fly.io/docs/security/tokens/)
to connect to the machines using SSH allowing me to send commands there in an automated way.

## Generating your access token

First step is to create an access token that allows me to send commands to my machine without requiring any
manual form of authentication. This can be done using the following command:

```bash
fly tokens create ssh -n my-token-name
```

Check the [command documentation](https://fly.io/docs/flyctl/tokens-create-ssh/) for more options. The output of 
this command will be as the following, where `<TOKEN_CONTENT_STRING>` will be a very long string that
you need to store and don't share it publicly.

```bash
FlyV1 <TOKEN_CONTENT_STRING>
```

Add the token to an environment var in the machine you will run the backup script:

```bash
export FLY_SSH_TOKEN=<TOKEN_CONTENT_STRING>
```

## Data location

The volume is mounted in `/data` directory, defined in our application `fly.toml` file:

```
[[mounts]]
  source = 'app_data'
  destination = '/data'
```

## Creating a backup script

Given that we have a token, we are now able to execute SSH commands remotely on our Fly Machine. In
our scenario, I am compacting the whole content of `/data/` directory (where all the data that I want
to backup is located) generating a tarball, then I download it locally.

I could create a custom script and copy it to the remote machine if I want to perform more complex
tasks, or you can extend/modify this script to perform other tasks (e.g. download the tarball
and upload to a S3 bucket).

```
#!/bin/bash
# backup.sh

# Need to start container in fly.io if it was stopped by inactivity
curl -s -o /dev/null https://your-app.fly.dev/

filename="data_backup_$(date +%F).tar.gz"
fly ssh console -C 'tar cvz /data' -t $FLY_SSH_TOKEN > $filename
```

## Run it periodically

Now you can add `backup.sh` to your `crontab` schedule, or even adapt the procedure described here
to be executed in other environments, like defining a GitHub Action or another way to schedule jobs.

I know this is not the most complete way to implement a backup policy, but it is working for my current projects.
In the future, as I improve my scripts, I will possibly update this post to make it more complete.
