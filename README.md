# GeoKretyMap
GeoKretyMap is a companion website to [geokrety.org](geokrety.org).

The most important services are:
* Interactive map of GeoKrety in the world
* A backend public Api that act as a cache/agreagation over [geokrety.org](geokrety.org).

# Installation
All components are packaged as a Docker containers.

There are 4 components to be installed:
* BaseX xml database (Api)
* Nginx webserver (Reverse proxy for API & Website)
* The main website (Website generator ; Jekyll)
* The cron (Launch updates regularly)

The `docker-compose.yml` present in this repository will start the full stack.

## docker-compose.yml
```
version: '2'
services:
  gkm-api-basex:
    image: geokretymap/gkm-api-basex
    container_name: gkm-api-basex
    volumes:
      - /srv/GKM/basex/data/BaseXData/:/srv/BaseXData/
    environment:
      - BASEX_JVM=-Xmx3072m
    restart: always

  gkm-website:
    image: geokretymap/gkm-website
    container_name: gkm-website
    volumes:
      - /srv/GKM/geokretymap-website/:/data/
    environment:
      - GKM_API_URL="https://api.<mydomain>"

  gkm-api-nginx:
    container_name: gkm-api-nginx
    image: geokretymap/gkm-api-nginx
    links:
      - gkm-api-basex:database
    volumes:
      - /srv/GKM/basex/data/BaseXData/:/var/www/html/basex/:ro
      - /srv/GKM/geokretymap-website/geokretymap.org/:/var/www/html/geokretymap.org/:ro
    restart: always

  #gkm-api-cron:
  #  container_name: gkm-api-cron
  #  image: geokretymap/gkm-api-cron
  #  restart: never
  #  links:
  #    - reverseproxy:api.gkm.kumy.org
```

## BaseX (Database)
BaseX is an xml database. We store all informations in:
* `geokrety`
* `geokrety-details`

There is 2 other bases for managing the update queue:
* `pending-geokrety`
* `pending-geokrety-details`

At first start, `BaseX` need to be configured and data imported.

Warning: geokrety-details are exported as distinct files. Ensure your partition has enought `inodes` or use a filesystem without `inodes` limit like `xfs`, `reiserfs`, `btrfs`...

### Change admin password
Even if basex ports are not exposed, it is a best practice to always change the root/admin password.

```
# docker exec -it gkm-api-basex basexclient
Username: admin
Password: admin
BaseX 8.4.3 [Client]
Try 'help' to get more information.
> alter password admin
Password: 
Password of user 'admin' changed.
```

### Import initial data
Databases need to be created and populated. Let's create them and use exports from GeoKretyMap.

```
# docker exec -it gkm-api-basex basexclient
Username: admin
Password: 
BaseX 8.4.3 [Client]
Try 'help' to get more information.

> run /srv/scripts/gkm-create.xq

Query "gkm-create.xq" executed in 105064.69 ms.

> run /srv/scripts/gkm-write-details.xq

Query "gkm-write-details.xq" executed in 52517.14 ms.

```

### Verify
You should now see the 4 databases:

```
> list
Name                      Resources  Size      Input Path                                                    
-----------------------------------------------------------------------------------------------------------
geokrety                  1          14578045  https://api.geokretymap.org/basex/export/geokrety_full-dump.xml  
geokrety-details          1          91500368  https://api.geokretymap.org/basex/export/geokrety-details.xml    
pending-geokrety          1          4785      pending-geokrety.xml                                          
pending-geokrety-details  1          4798      pending-geokrety-details.xml                                  

4 database(s).
```

## Jekyll (Website)
The website is generated using Jekyll. This container just generate the content as static files in a shared directory. The static files need to be served using `Nginx` here.

## Nginx (Webserver)
Nginx is used here as a reverse proxy for the api and standard webserver for the website ; 2 vhosts.

### default vhost
The website is server by the default vhost.

The container accept the `GKM_API_URL` environment variable. Use it to match your current dev environment or your mirror.

Don't forget to adapt the `docker-compose.yml` file!

### api vhost
To be accessible, this vhost must start by `api.` (ex: `api.geokretymap.org` or `api.gkm.kumy.org`). The used url must match what defined in `GKM_API_URL`.

## Cron
The `gkm-api-cron` container is responsible for calling some private admin urls.

This is responsible for maintaining up to date data, or schedule backup and exports.
