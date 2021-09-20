# Joomla! 

This is the **octoleo** [images of Joomla!](https://hub.docker.com/r/llewellyn/joomla) maintained by @llewellynvdm mostly to try out new ideas, you are welcome to give them a try. Feedback is also welcome ʕ•ᴥ•ʔ

**This is not a stable place, as I plan to try out new ideas for the [official Joomla-Docker](https://github.com/joomla-docker/docker-joomla) images here.**

# Docker Compose
setup [docker compose](https://docs.docker.com/compose/install/)

### Setup Traefik
> I usually add [traefik](https://doc.traefik.io/traefik/) as my revers proxy to its own docker-compose.yml file in its own folder like: **/home/username/Docker/traefik**
```yml
version: "3.3"

services:
  traefik:
    container_name: traefik
    image: "traefik:latest"
    command:
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
#      - --api.dashboard=true
#      - --api.insecure=true
      - --providers.docker
      - --log.level=ERROR
      - --certificatesresolvers.octoleoresolver.acme.httpchallenge=true
      # change this email address to your own
      - --certificatesresolvers.octoleoresolver.acme.email=joomla@yourdomain.com
      - --certificatesresolvers.octoleoresolver.acme.storage=/acme.json
      - --certificatesresolvers.octoleoresolver.acme.httpchallenge.entrypoint=web
      - --providers.file.directory=/conf
      - --providers.file.watch=true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      # You must create these paths before the : on the host system
      - "/home/username/Docker/traefik/conf:/conf"
      - "/home/username/Docker/traefik/acme.json:/acme.json"
      - "/home/username/Docker/traefik/errors:/errors"
    labels:
      # settings for all containers
      - "traefik.http.routers.http-catchall.rule=hostregexp(`{host:.+}`)"
      - "traefik.http.routers.http-catchall.entrypoints=web"
      - "traefik.http.routers.http-catchall.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
    networks:
      - traefik

networks:
  traefik:
    external:
      name: traefik_webgateway
```
Then run the following command in the folder where you placed the docker-compose.yml file:
```shell
docker-compose up -d
```

## Setup Joomla
> Here I will just setup Joomla v3.10, but you can just change the version with any of the following tags:

- `joomla:4.0`
- `joomla:3.10`
- `joomla:3.9`

> The following setup is usually placed in its own docker-compose.yml file in its own folder like: **/home/username/Docker/websitename**

```yml
version: '2'
services:
  mariadb_websitename:
    image: mariadb:latest
    container_name: mariadb_websitename
    restart: unless-stopped
    environment:
      - MARIADB_USER=octoleo
      - MARIADB_DATABASE=octoleo
      - MARIADB_PASSWORD=XXXXXXXXXXXXX
      - MARIADB_ROOT_PASSWORD=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    volumes:
      # to make your files persistent and acceptable on the host system
      - '/home/username/Projects/websitename/db:/var/lib/mysql'
    networks:
      - traefik

  joomla_websitename:
    image: llewellyn/joomla:3.10
    container_name: joomla_websitename
    restart: unless-stopped
    environment:
      - JOOMLA_DB_HOST=mariadb_websitename:3306
      - JOOMLA_DB_USER=octoleo
      - JOOMLA_DB_NAME=octoleo
      - JOOMLA_DB_PASSWORD=XXXXXXXXXXXXX
    depends_on:
      - mariadb_websitename
    volumes:
      # to make your files persistent and acceptable on the host system
      - '/home/username/Projects/websitename/website:/var/www/html'
    networks:
      - traefik
    labels:
      # joomla
      - "traefik.enable=true"
      # the website domain pointing to this server
      - "traefik.http.routers.joomla_websitename.rule=Host(`octoleo.yourdomain.com`)"
      - "traefik.http.routers.joomla_websitename.entrypoints=websecure"
      - "traefik.http.services.joomla_websitename.loadbalancer.server.port=80"
      - "traefik.http.routers.joomla_websitename.service=joomla_websitename"
      - "traefik.http.routers.joomla_websitename.tls.certresolver=octoleoresolver"\

  # only if you would like to access the DB via phpmyadmin
  # else you can remove this and all will still work
  phpmyadmin_websitename:
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin_websitename
    restart: unless-stopped
    environment:
      PMA_HOST: mariadb_websitename
      PMA_PORT: 3306
      UPLOAD_LIMIT: 300M
    depends_on:
      - mariadb_websitename
    networks:
      - traefik
    labels:
      # joomla
      - "traefik.enable=true"
      # the database domain pointing to this server
      - "traefik.http.routers.phpmyadmin_websitename.rule=Host(`octoleo-db.yourdomain.com`)"
      - "traefik.http.routers.phpmyadmin_websitename.entrypoints=websecure"
      - "traefik.http.services.phpmyadmin_websitename.loadbalancer.server.port=80"
      - "traefik.http.routers.phpmyadmin_websitename.service=phpmyadmin_websitename"
      - "traefik.http.routers.phpmyadmin_websitename.tls.certresolver=octoleoresolver"

networks:
  traefik:
    external:
      name: traefik_webgateway
```
Then run the following command in the folder where you placed the docker-compose.yml file:
```shell
docker-compose up -d
```

## Usecase

I often like to run a few Joomla instance (docker containers) on the same server, and be able to access them easy without the pain of configuring proxy stuff, setup SSL or any other tedious configurations.

- So I use [Cloudflare](https://cloudflare.com/) to point my domain as wildcard to my server:
![image](https://user-images.githubusercontent.com/5607939/133998478-5c2b5a3d-bea1-4336-88a8-e102176f2abb.png)
- Then I create two main folder on the host system in my user directory **Docker** and **Projects**.
- The **Docker** folder is used to store each project's docker-compose.yml file in its own sub folder.
- The **Projects'** folder is used whenever I want the project to have persistent and/or acceptable files on the host system.

The above [Traefik](https://github.com/octoleo/docker-joomla#setup-traefik) configurations needs only be done once. But you can add as many [Joomla containers](https://github.com/octoleo/docker-joomla#setup-joomla) as you like, just update the `websitename` placeholder in the docker-compose.yml configurations then place this new docker-compose.yml in a sub folder (_websitename_) inside the **Docker** folder, and run `docker-compose up -d` inside this sub folder.

Then give it a moment to get everything ready... and when you open your domain, the ssl should already be active, and Joomla ready to install via the browser.

> Have Fun!!

### Free Software License
```txt
@copyright  Copyright (C) 2021 Llewellyn van der Merwe. All rights reserved.
@license    GNU General Public License version 2 or later; see LICENSE.txt
```
