---
title: "Migrate sameersbn/docker-gitlab to gitlab/gitlab-ce"
slug: "migrate-sameersbn-docker-gitlab-to-gitlab-gitlab-ce"
date: 2018-06-12T17:15:55+08:00
draft: false
tags: ["docker", "gitlab"]
image: "/content/images/2018/06/DSC03526.jpg"
---

This article records how I migrate [GitLab](https://about.gitlab.com/) from [sameersbn/gitlab](https://github.com/sameersbn/docker-gitlab) to [gitlab/gitlab-ce](https://hub.docker.com/r/gitlab/gitlab-ce/).

# Why
Why would I bother to migrate it? There are several reasons and here are the most important (for me) ones.

- If you want to enable some functions in GitLab, you would search it from official documents and here comes the pain: you don’t know how to configure it in `sameersbn/gitlab`. Though the documents of sameersbn/gitlab are quite good, it's sometimes hard to mapping them together
- `sameersbn/gitlab` is driven by  community and you don’t know when it would give up maintaining it (for the record, `sameersbn/docker-gitlab` is a excellent project. Well documented and always keep up the releases with official GitLab)
- I love official thing

# What
What are the problems we would encounter? There is one major problem: the installation source are different between `sameersbn/docker-gitlab` and `gitlab/gitlab-ce`. The former install GitLab from source and the latter install it from omnibus package (it means Ubuntu or Debian package, I spent nearly half year to figure it out) which makes it much more difficult to migrate.

# How
I was trying to map the data folder between `sameersbn/docker-gitlab` and `gitlab/gitlab-ce` and it failed, of course. Someday the idea just came up to my mind: why not just backup GitLab and restore it? GitLab must have provided the backup and restore functions and it should be the same format no matter how you install GitLab from. It works but there is another problem now: some secret configuration files are excluded from the backup. Now I can't access the secret variables and database because I don't have the keys. I'd had few tries to covert the configuration file and it works as I thought.  

For simplicity, all paths below we talk about are the paths in container. If I mention about transferring the files between two containers, you may use the command `docker cp` to copy the files to the host and then copy it to the other container. Or, you may mount the folder to the host and simply copy the files.

Here are the steps how I migrate it and the details are at the bottom.
1. Create backup from `sameersbn/docker-gitlab` container
2. Spin up a new official GitLab container
3. Change the configuration in official GitLab container
4. Restore from backup
5. Check

## Create Backup From sameersbn/docker-gitlab Container
Mapping the data by folder between them is not a good idea so the easiest way is that use the built-in backup and restore functions.

1. Shut down GitLab, redis and postgresql containers
2. Spin up GitLab and other services again but make GitLab unreachable via not exposing the port
3. Access GitLab container inside and execute the command: `/sbin/entrypoint.sh app:rake gitlab:backup:create`
4. Backup file will be generated at `/home/git/data/backups` in container

Reference: https://github.com/sameersbn/docker-gitlab#creating-backups

## Spin Up an Official GitLab Container

We will have to change the configuration for storing our secrete keys otherwise we can’t access the secret variables or database. We first spin up an official GitLab container to initialize the folder structure so that we can easily find the configuration files.

1. Create 3 folders first: `config, logs, data`
2. Execute the command and remember choose the same verion of GitLab:

```
docker run -it \
    -p 10080:80 \
    -p 10022:22 \
    --name gitlab \
    -v /PATH-TO-YOUR-FOLDER/config:/etc/gitlab:Z \
    -v /PATH-TO-YOUR-FOLDER/logs:/var/log/gitlab:Z \
    -v /PATH-TO-YOUR-FOLDER/data:/var/opt/gitlab:Z \
    gitlab/gitlab-ce:YOUR-GITLAB-VERSION
```
3. Wait until all components have been initialized. You can hit the home page of GitLab via `http://YOUR-IP:10080/` to check. When you see the page asks you to set new password for `root`, it means all components are done.
4. Copy the backup file to `/var/opt/gitlab/backups/` in official GitLab container

Reference: https://docs.gitlab.com/omnibus/docker/

## Change the Configuration in Official GitLab Container

There are some secret keys for encrypting the data in GitLab and the built-in backup function won’t backup those things due to security reason. We must change it by ourselves and here are the instructions:

1. Stop the official GitLab container
2. Find out these 3 keys in `sameersbn/docker-gitlab` container:
- `GITLAB_SECRETS_OTP_KEY_BASE`
- `GITLAB_SECRETS_DB_KEY_BASE`
- `GITLAB_SECRETS_SECRET_KEY_BASE`

You can take a look in your start script or access the `sameersbn/docker-gitlab` container to see the file: `/home/git/gitlab/config/secrets.yml`

4. Fill those 3 keys to the corresponding field in the file `/etc/gitlab/gitlab-secrets.json` of official GitLab container
5. Copy all other files under `/home/git/data/ssh/*` in `sameersbn/docker-gitlab` container to the config location of official GitLab, which is `/etc/gitlab/`
6. Spin up the official GitLab container again

Reference: https://docs.gitlab.com/ce/raketasks/backup_restore.html#restore-prerequisites

## Restore
1. Access the container of official GitLab: `docker exec -ti gitlab /bin/bash`
2. Stop `unicorn` and `sidekiq`:
```
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
# Verify
gitlab-ctl status
```
3. Restore GitLab and it will ask you some questions to double confirm the restoring process because it will delete all contents of your GitLab database. You also must specify the timestamp of the backup you wish to restore:
```
# This command will overwrite the contents of your GitLab database!
gitlab-rake gitlab:backup:restore BACKUP=1493107454_2018_04_25_10.6.4-ce
```
4. Restart GitLab and check:
```
gitlab-ctl restart
gitlab-rake gitlab:check SANITIZE=true
```

Reference: https://docs.gitlab.com/ce/raketasks/backup_restore.html#restore-for-omnibus-installations


## Check

There is no script to do this and you could probably only try hard to click every page on GitLab. Good luck.  
The best way I could come up with is that writing a script to access every project in the GitLab. If the response code is `200`, GitLab probably works fine.

# Notes
- I guess you can restore the data and then changing the config (`/etc/gitlab/gitlab-secrets.json`) since the official restoring document says so.
- It’s very hard to migrate and you definitely won’t want to use production data. Use backup data to practice.