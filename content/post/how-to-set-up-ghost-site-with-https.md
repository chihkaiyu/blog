---
title: "How to Install Ghost with Nginx and MySQL"
slug: "how-to-set-up-ghost-site-with-https"
date: 2018-05-19T23:33:58+08:00
draft: false
tags: ["docker"]
image: "/content/images/2018/05/DSC03682.jpg"
---


This article will show you how to install and configure [Ghost](https://ghost.org/) with [Nginx](https://www.nginx.com/) and [MySQL](https://www.mysql.com/).
In fact, it is that how this blog has been built.
This article contains:
- [Prerequisite](#prerequisite)
- [Environment](#environment)
- [Ghost configuration](#ghost)
- [Nginx configuration](#nginx)
- [MySQL configuration](#mysql)
- [Docker Compose](#docker-compose)
- [Summary](#summary)

<a name="prerequisite"></a>
# Prerequisite
I use [Docker](https://www.docker.com/) for handling my environment information and I suppose you have the basic knowledge about Docker and Docker Compose.
Before you continue, you must ensure both of Docker and [Docker Compose](https://docs.docker.com/compose/overview/) have been installed on your machine.
Here are official guide to install them:
- [Docker installation guide](https://docs.docker.com/install/#cloud)
- [Docker Compose installation guide](https://docs.docker.com/compose/install/)

<a name="environment"></a>
# Environment
Here is my environment information:
- OS: `Ubuntu 16.04.2 LTS`
- Docker version: `17.04.0-ce`
- Docker Compose version: `1.10.1`

The image version of other components will be introduced below.

<a name="ghost"></a>
# Ghost Configuration
Since we run Ghost in Docker container, the best way to configure it is via environment variables. You'll have to [insert two underscores](https://docs.ghost.org/docs/config#section-running-ghost-with-config-env-variables) for nested configurations. You can see the examples below.
You can also generate the configuration file in json format. Read the [official documents](https://docs.ghost.org/docs/config) for more information.
Here is my Ghost configuration in `docker-compose.yml`:
```
environment:
  url: https://blog.kaiyu.site
  admin__url: https://blog.kaiyu.site
  database__client: mysql
  database__connection__host: ghost-db
  database__connection__port: 3306
  database__connection__user: {USER}
  database__connection__password: {PASSWORD}
  database__connection__database: {DATABASE}
```

<a name="nginx"></a>
# Nginx Configuration
I use Nginx as a reverse proxy for easier configuring HTTP behavior like cache period or SSL settings.
The reason I want to configure HTTP server is that I want to install SSL certificate and it's very easy via Nginx.
I apply SSL certificate on [SSL For Free](https://www.sslforfree.com/) and it's totally free but the problem is that you'll have to renew your certificate every 3 months.
I purchased my domain on [GoDaddy](https://www.godaddy.com/).
Here is my Nginx configuration in `nginx/blog.conf`:
```
server {  
  listen              80;
  server_name         blog.kaiyu.site;
  return              301 https://$server_name$request_uri;
}

server {  
  listen              443;
  server_name         blog.kaiyu.site;
  access_log          /var/log/nginx/blog.kaiyu.site.log;

  ssl on;
  ssl_certificate      /etc/ssl/blog.kaiyu.site/bundle.crt;
  ssl_certificate_key  /etc/ssl/blog.kaiyu.site/private.key;

  ssl_session_timeout 5m;
  ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;

  location / {
    proxy_set_header  Host                $host;
    proxy_set_header  X-Forwarded-Proto   $scheme;
    proxy_set_header  X-Real-IP           $remote_addr;
    proxy_set_header  X-Forwarded-For     $proxy_add_x_forwarded_for;
    proxy_http_version                    1.1;
    proxy_connect_timeout                 90;
    proxy_send_timeout                    90;
    proxy_read_timeout                    90;
    proxy_buffer_size                     4k;
    proxy_buffers                         4 32k;
    proxy_busy_buffers_size               64k;
    proxy_temp_file_write_size            64k;

    client_max_body_size                  10m;
    client_body_buffer_size               128k;

    proxy_pass                            http://ghost:2368;
  }
}
```

<a name="mysql"></a>
# MySQL Configuration
I'd only set up the password of root, database, user, the password of user via environment variables.
The initialization is all done via Ghost.
Here is my MySQL configuration in `docker-compose.yml`:
```
environment:
  MYSQL_ROOT_PASSWORD: {ROOT-PASSWORD}
  MYSQL_DATABASE: {DATABASE}
  MYSQL_USER: {USER}
  MYSQL_PASSWORD: {PASSWORD}
```

<a name="docker-compose"></a>
# Docker Compose
Finally you can sum up everything in `docker-compose.yml` and spin them up via a single command: `docker-compose up -d`.
The Docker image tag of component I've used are:
- Ghost: `1.22.7-alpine`
- Nginx: `1.13.12-alpine`
- MySQL: `5.7.22`

The important thing is persisting your data by simply mounting the data outside to the host.
Here are the locations of every component's data:
- Ghost: `/var/lib/ghost/content`
- MySQL: `/var/lib/mysql`

Here is my `docker-compose.yml`:
```
version: '2.1'

services:
  nginx:
    image: nginx:1.13.12-alpine
    restart: always
    depends_on:
      - ghost
      - ghost-db
    ports:
      - 80:80
      - 443:443
    volumes:
      - $PWD/nginx/blog.conf:/etc/nginx/conf.d/default.conf
      - $PWD/ssl:/etc/ssl/blog.kaiyu.site
  ghost:
    image: ghost:1.22.7-alpine
    restart: always
    depends_on:
      - ghost-db
    environment:
      url: https://blog.kaiyu.site
      admin__url: https://blog.kaiyu.site
      database__client: mysql
      database__connection__host: ghost-db
      database__connection__port: 3306
      database__connection__user: {USER}
      database__connection__password: {PASSWORD}
      database__connection__database: {DATABASE}
      VIRTUAL_HOST: blog.kaiyu.site
    volumes:
      - $PWD/ghost:/var/lib/ghost/content

  ghost-db:
    image: mysql:5.7.22
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: {ROOT-PASSWORD}
      MYSQL_DATABASE: {DATABASE}
      MYSQL_USER: {USER}
      MYSQL_PASSWORD: {PASSWORD}
    volumes:
      - $PWD/mysql:/var/lib/mysql

```

<a name="#summary"></a>
# Summary
In this article, you have learned how to install Ghost with Nginx and MySQL.
All these things can be done very easily via Docker.
With Docker, the environment is not a problem nowadays but you'll still need to figure out how to configure every component.