# Nextcloud on Docker

This repository contains a Docker-based Nextcloud setup with LetsEncrypt SSL, PostgreSQL backend, Collabora Online Office and preview generation.

##  Credits

The configuration is based on [this official full example](https://github.com/nextcloud/docker/blob/ac8c9984319e45fd34fa3863f82fd9063d628aa4/.examples/dockerfiles/full/apache/Dockerfile), [bentolor's setup](https://github.com/bentolor/docker-nextcloud-collabora-postgresql-letsencrypt) as well as [Gerald Allerstorfer's](https://www.allerstorfer.at/nextcloud-install-preview-generator/) guide for enabling preview generation.

### Difference

My setup differs from bentolor's setup by using a slightly modified version of the official full example. In particular, I've modified this example to install the `libreoffice` and `ghostscript` packages in order to make preview generation work for LibreOffice documents and PDF files, respectively.

## Prerequisites

This Nextcloud setup requires docker-compose and the instructions assume that your host machine is running a Linux distribution. In addition, you must

- register two domains, one for Nextcloud and one for Collabora. I personally use https://duckdns.org but there are many other options out here.
- Publicly expose port 80 and 443 in your router.
- Adjust all values enclosed by `[[##....##]]` in the `docker-compose.yml` and `db.env` files.

## Install

The `docker-compose.yml` file and the following instructions assume the Nextcloud data will be stored in `/mnt/md0/nc-data/` on the host machine, so make sure you update this location to match your needs.

First, we will create the data directory for the Nextcloud data and set the permissions:

```bash
$ sudo mkdir /mnt/md0/nc-data && \
       touch /mnt/md0/nc-data/.ocdata && \
       chown -R www-data:www-data /mnt/md0/nc-data/
```

Next, build and run the containers:

```bash
$ docker-compose up -d --build
```

Once your Nextcloud instance is up running  (confirm this by accessing your Nextcloud domain), you'll need to execute the following command to make your Nextcloud instance work with the Android app or other desktop clients (such as the one available for Ubuntu).

``` bash
$ docker exec -u www-data nextcloud_app_1 php occ config:system:set overwriteprotocol --value="https"
```

In this command (and the ones that follow!), `nextcloud_app_1` is the name of the Nextcloud container. As pointed out in bentolor's example, executing this command is necessary until [this pull request](https://github.com/nextcloud/docker/pull/819) has been merged. The issue is described in more detail [here](https://github.com/nextcloud/android/issues/4786).

### Enable Collabora

To enable Collabora, log in to your Nextcloud using the admin credentials specified in your `docker-compose.yml` file and

- install the Collabora plugin from the Nextcloud app store:
  - Apps -> Download and enable "Collabora Online".
- Finally, configure "Collabora Online" to point to your Collabora domain:
  - Settings -> Administration -> Collabora Online -> Enter the URL for your Collabora domain -> press Apply.

### Enable Preview Generation

To get preview generation to work, install the Preview Generator from the Nextcloud app and add the following to your `config.php`

``` 
'preview_libreoffice_path' => '/usr/bin/libreoffice',
'enable_previews' => true,
'enabledPreviewProviders' =>
 array (
    0 => 'OC\\Preview\\TXT',
    1 => 'OC\\Preview\\MarkDown',
    2 => 'OC\\Preview\\OpenDocument',
    3 => 'OC\\Preview\\PDF',
    4 => 'OC\\Preview\\MSOffice2003',
    5 => 'OC\\Preview\\MSOfficeDoc',
    6 => 'OC\\Preview\\PDF',
    7 => 'OC\\Preview\\Image',
    8 => 'OC\\Preview\\Photoshop',
    9 => 'OC\\Preview\\TIFF',
   10 => 'OC\\Preview\\SVG',
   11 => 'OC\\Preview\\Font',
   12 => 'OC\\Preview\\MP3',
   13 => 'OC\\Preview\\Movie',
   14 => 'OC\\Preview\\MKV',
   15 => 'OC\\Preview\\MP4',
   16 => 'OC\\Preview\\AVI',
 ),
```

One way to do this, is by accessing `config.php` from your host machine (note that the name of your Docker volume might be slightly different):

``` bash
$ sudo nano /var/lib/docker/volumes/nextcloud_nextcloud-data/_data/config/config.php
```

Next, generate previews for all files:

``` bash
$ docker exec -u www-data nextcloud_app_1 php occ preview:generate-all
```

Finally, we will add a cron job that generates previews for new or modified files. One way to do this is by creating a shell script that performs this task:

``` bash
#!/bin/sh
docker exec -u www-data nextcloud_app_1 php occ preview:pre-generate
```

To run this script (`~/nextcloud/pre-generate.sh`) every 10 minutes, execute `crontab -e` and add the following to the current crontab:

``` bash
*/10 * * * * ~/nextcloud/pre-generate.sh
```

That's it!

### Setting up SMTP

You may want to set up SMTP to enable email notifications. Although this can partly be achieved by editing the `docker-compose.yml` file, you can also do this by following the instructions in [this guide](https://www.techrepublic.com/article/how-to-configure-smtp-for-nextcloud/).

## Upgrade

To upgrade your installation, execute the following commands.

```bash
$ docker-compose down
$ docker-compose build --pull
$ docker-compose up -d
```

Although I haven't had the need to do this yet, it may be necessary to execute `docker exec --user www-data nextcloud_app_1 php occ db:add-missing-indices` after a major release upgrade (as pointed out by bentolor).

## Migrate PostgreSQL data

The example below shows how I used tianon's Docker image to upgrade my PostgreSQL data from version 12 to 13. The `_data` folder, referred to in the example, contains the PostgreSQL data being migrated. In my case the `_data` folder is contained in `/var/lib/docker/volumes/nextcloud_nextcloud-db/`.

Before you try this make sure you backup your PostgreSQL data.

First, prepare your data for migration.

```bash
$ mkdir -p /migrate/12
$ mkdir -p /migrate/13
$ cp -rp _data /migrate/12/data
$ cd migrate
```
Next, perform the migration (note that your paths may differ).

```bash
$ docker run --rm -e PGUSER=nextcloud -e POSTGRES_INITDB_ARGS="-U nextcloud" -v /var/lib/docker/volumes/nextcloud_nextcloud-db/migrate/:/var/lib/postgresql tianon/postgres-upgrade:12-to-13
```

After that copy over your client authentication configuration to the new data:

```bash
cp -rp 12/data/pg_hba.conf 13/data/pg_hba.conf
```

Finally replace your old data with the new data:

```bash
$ rm -rf /var/lib/docker/volumes/nextcloud_nextcloud-db/_data
$ cp -rp 13/data /var/lib/docker/volumes/nextcloud_nextcloud-db/_data
```
