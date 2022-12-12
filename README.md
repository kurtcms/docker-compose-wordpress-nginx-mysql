# Docker Compose: WordPress on MySQL and NGINX with Certbot

This multi-container Docker app is orchestrated with [Docker Compose](https://docs.docker.com/compose/) for rapid and modular deployment that fits in any microservice architecture.

It creates a [WordPress](https://wordpress.com/) website on a [MySQL](https://www.mysql.com/) database and an [NGINX](https://www.nginx.org/) web server, with [Certbot](https://certbot.eff.org/) by the [Electronic Frontier Foundation](https://www.eff.org/) (EFF) for obtaining and renewing a SSL/TLS certificate on a given root domain from [Let’s Encrypt](https://letsencrypt.org/), a non-profit Certificate Authority by the [Internet Security Research Group](https://www.abetterinternet.org/) (ISRG).

A detailed walk-through is available [here](https://kurtcms.org/docker-compose-wordpress-on-mysql-and-nginx-with-certbot/).

## Table of Content

- [Getting Started](#getting-started)
  - [Git Clone](#git-clone)
  - [Environment Variable](#environment-variables)
  - [Docker Compose](#docker-compose)
    - [Install](#install)
    - [SSL/TLS Certificate](#ssltls-certificate)
    - [Up and Down](#up-and-down)
- [Backup and Restore](#backup-and-restore)
    - [Docker Volume](#docker-volume)
    - [MySQL](#mysql)
    - [WordPress](#wordpress)
- [SSL/TLS Certificate Renewal](#ssltls-certificate-renewal)
- [Reference](#reference)

## Getting Started

Get started in three simple steps:

1. [Download](#git-clone) a copy of the app;
2. Create the [environment variables](#environment-variables) for the MySQL root password and the WordPress database settings; and
3. [Docker Compose](#docker-compose) to start the app.

### Git Clone

Download a copy of the app with `git clone`. Be sure to pass the `--recurse-submodules` argument to initialise and update each submodule in the repository.

```shell
$ git clone --recurse-submodules https://github.com/kurtcms/docker-compose-wordpress-nginx-mysql /app/docker-compose-wordpress-nginx-mysql/
```

### Environment Variables

Docker Compose expects the MySQL root password, the WordPress database name, username and password as environment variables in a `.env` file in the same directory.

Be sure to create the `.env` file.

```shell
$ nano /app/docker-compose-wordpress-nginx-mysql/.env
```

And define the variables accordingly.

```
# MySQL root password
MYSQL_ROOT_PASSWORD='(redacted)'

# WordPress database name, username and password
MYSQL_WORDPRESS_DATABASE='kurtcms_org'
MYSQL_WORDPRESS_USER='kurtcms_org'
MYSQL_WORDPRESS_PASSWORD='(redacted)'
```

### Docker Compose

With Docker Compose, the app may be provisioned with a single command.

#### Install

Install [Docker](https://docs.docker.com/engine/install/) and [Docker Compose](https://docs.docker.com/compose/install/) with the [Bash](https://github.com/gitGNU/gnu_bash) script that comes with app.

```shell
$ chmod +x /app/docker-compose-wordpress-nginx-mysql/docker-compose/docker-compose.sh \
    && /app/docker-compose-wordpress-nginx-mysql/docker-compose/docker-compose.sh
```

#### SSL/TLS Certificate

A dummy SSL/TLS certificate and private key will need to be created for NGINX to start service, and for Certbot to subsequently obtain a signed SSL/TLS certificate from Let’s Encrypt by answering a [HTTP-01 challenge](https://letsencrypt.org/docs/challenge-types/#http-01-challenge).

Create the dummy certificate before obtaining a signed one with the Bash script that comes with app.

```shell
$ chmod +x /app/docker-compose-wordpress-nginx-mysql/certbot/certbot.sh \
    && /app/docker-compose-wordpress-nginx-mysql/certbot/certbot.sh
```

Enter the root domain and the email address for registration and recovery when prompted.

#### Up and Down

Start the containers with Docker Compose.

```shell
$ docker-compose up -d
```

Stopping the containers is as simple as a single command.

```shell
$ docker-compose down
```

## Backup and Restore

With the Docker containers up and running, WordPress will be reachable at the localhost address on TCP port 443 i.e. HTTPS, to which TCP port 80 i.e. HTTP will be redirected. Follow the instructions on screen to set up WordPress. Alternatively the MySQL database and WordPress content can be restored from an existing backup.

### Docker Volume

The MySQL database, the WordPress content as well as the SSL/TLS certificate and its corresponding private key are stored in separate `Docker volume` under `/var/lib/docker/volumes/`.

```
/var/
└── lib/
    └── docker/
        └── volumes/
            ├── docker-compose-wordpress-nginx-mysql_certbot/
            ├── docker-compose-wordpress-nginx-mysql_mysql/
            └── docker-compose-wordpress-nginx-mysql_wordpress/
```

### MySQL

MySQL comes with a tool for [logical backup](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_logical_backup). It produces a set of SQL statements that can be executed to reproduce the original database object definitions and table data.

Back up an existing MySQL database in a container.

```shell
$ docker exec -it mysql mysqldump \
    --user root \
    --password \
    kurtcms_org > kurtcms_org.sql
```

Restore a database from an existing backup.

```shell
$ docker exec -it mysql mysql \
    --user root \
    --password \
    kurtcms_org < kurtcms_org.sql
```

### WordPress

WordPress plugins and themes, as well as media files uploaded by users are housed under `wp-content` in the WordPress directory. Backing up and restoring them is as simple as making a copy of this directory and moving it to the WordPress container.

Making a copy of the `wp-content` directory.

```shell
$ cp -r /var/lib/docker/volumes/docker-compose-wordpress-nginx-mysql_wordpress/_data/wp-content \
    /app/docker-compose-wordpress-nginx-mysql/
```

Restoring it from an existing backup.

```shell
$ cp -r /app/docker-compose-wordpress-nginx-mysql/wp-content/ \
    /var/lib/docker/volumes/docker-compose-wordpress-nginx-mysql_wordpress/_data/
```

## SSL/TLS Certificate Renewal

Certbot is instructed by Docker Compose to attempt a SSL/TLS certificate renewal every 12 hours, which should be more than adequate considering the certificate is [valid for 90 days](https://letsencrypt.org/docs/faq/#what-is-the-lifetime-for-let-s-encrypt-certificates-for-how-long-are-they-valid).

NGINX is instructed to reload its configuration every 24 hours to ensure the renewed certificate will come into effect at most 12 hours after a renewal, which should also be well in advance of an impending expiry.

Edit the `docker-compose.yml` should these intervals need to be adjusted.

```shell
$ nano /app/docker-compose-wordpress-nginx-mysql/docker-compose.yml
```

Modify the values as appropriate.

```
  nginx:
    ...
    command: "/bin/sh -c 'while :; do sleep 24h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"

  certbot:
    ...
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

## Reference

- [Docker Hub - MySQL](https://hub.docker.com/_/mysql)
- [Docker Hub - WordPress](https://hub.docker.com/_/wordpress)
- [Docker Hub - NGINX](https://hub.docker.com/_/nginx)
- [Docker Hub - Certbot](https://hub.docker.com/r/certbot/certbot)