# Run Matomo on Dokku

Deploy your own instance of [Matomo](https://matomo.org) on
[Dokku](https://github.com/dokku/dokku).

This setup is makes use of the great ready-to-use
[`matomo-docker`](https://github.com/crazy-max/docker-matomo) image by [crazy-max](https://github.com/crazy-max),
which does all of the heavy-lifting to properly deploy Matomo without too much headache.

## What you get

We will deploy [Matomo 3.13.5](https://github.com/matomo-org/matomo/releases/tag/3.13.5) onto your own Dokku server.

# Requirements

- [Dokku](https://github.com/dokku/dokku)
- [dokku-mariadb](https://github.com/dokku/dokku-mariadb)
- [dokku-letsencrypt](https://github.com/dokku/dokku-letsencrypt)

# Setup

## Pull image and tag it

We pull the `matomo` image from Docker Hub and tag it properly for Dokku.

```
docker pull crazymax/matomo:latest
docker tag crazymax/matomo:latest dokku/matomo:v3.13.5
```

## App and database

First create a new Dokku app. We will call it `matomo`.

```
dokku apps:create matomo
```

Next create the MariaDB database required by Matomo.

```
dokku mariadb:create mariadb-matomo
```

## Configuration

## Main configuration

```
dokku config:set --no-restart matomo TZ=Europe/Berlin
dokku config:set --no-restart matomo MEMORY_LIMIT=256M
dokku config:set --no-restart matomo UPLOAD_MAX_SIZE=16M
dokku config:set --no-restart matomo OPCACHE_MEM_SIZE=128
dokku config:set --no-restart matomo REAL_IP_FROM=0.0.0.0/32
dokku config:set --no-restart matomo REAL_IP_HEADER=X-Forwarded-For
dokku config:set --no-restart matomo LOG_LEVEL=WARN
```

## Persistent storage

You need to mount a volume on your host (the machine running Dokku) to persist all settings that you set in the Matomo interface.

```
mkdir /var/lib/dokku/data/storage/matomoo
# UID:GUID are set to 101. These are the values the nginx image uses,
# that is used by crazymax/matomo
chown 101:101 /var/lib/dokku/data/storage/matomo
dokku storage:mount matomo /var/lib/dokku/data/storage/matomo:/data
```

## Domain setup

To get the routing working, we need to apply a few settings. First we set the domain.

```
dokku domains:set matomo matomo.example.com
```

We also need to update the ports set by Dokku.

```
dokku proxy:ports-add matomo http:80:8000
dokku proxy:ports-remove matomo http:80:5000
```

If Dokku proxy:report sentry shows more than one port mapping, remove all port mappings except the added above.

## Email settings (optional)

You need to set the following settings if you want to receive emails from Matomo.

```
dokku config:set --no-restart SSMTP_HOST=smtp.example.com
dokku config:set --no-restart SSMTP_PORT=587
dokku config:set --no-restart SSMTP_HOSTNAME=matomo.example.com
dokku config:set --no-restart SSMTP_USER=user@example.com
dokku config:set --no-restart SSMTP_PASSWORD=yoursmtppassword
dokku config:set --no-restart SSMTP_TLS=YES
```

## Advanced configuration

If needed, the Matomo configuration file is located at `/var/lib/dokku/data/storage/matomo/config/config.ini.php` and can be manually edited.

# Deploy

## Deploy app for the first time

Deploy Matomo from the previously tagged docker image.

```
dokku tags:deploy matomo v3.13.5
```

## Setup Let's Encrypt

Setup an SSL certificate via Let's Encrypt.

```
dokku config:set --no-restart matomo DOKKU_LETSENCRYPT_EMAIL=letsencrypt@example.com
dokku letsencrypt matomo
dokku letsencrypt:auto-renew matomo
```

# Grep MariaDB information for the setup

We will need to set up Matomo in the web interface and provide the database details. You should be able to access the page via [`https://matomo.example.com`](https://matomo.example.com).

Run the command below to retrieve the DSN.

```
dokku mariadb:info mariadb-matomo
```

An example DSN might look like this: `mysql://mariadb:ffd4fc238ba8adb3@dokku-mariadb-mariadb-matomo:3306/mariadb_matomo`. Copy and paste the details as follows:

```
Hostname: dokku-mariadb-mariadb-matomo
Username: mariadb
Password: ffd4fc238ba8adb3
Database Name: mariadb_matomo
```

After going through the setup, you should be able to use Matomo.
