---
layout: post
title: Running Laravel inside Docker, on a Raspberry Pi
---

I have another post about running Laravel inside docker, which you can find [here](https://agjmills.github.io/laravel-docker-tutorial/)

With the recent release of the Raspberry Pi 4, with its ever increasing capabilities, Raspberry Pis are quickly becoming a real contender 
for a daily development machine. 

I recently trialed it, running [Ubuntu 20.04 for ARM](https://ubuntu.com/download/server/arm) with [Xfce](https://xfce.org/), and had to make
a few changes to my local development environment in order to accomodate the ARM processor. 

Many docker images are architecture specific - generally targetting modern 64bit processors, but these are different to ARM processors, therefore
in some cases, you may need to replace certain images with ARM specific containers.

Below is my standard `docker-compose.yml` for a Laravel application:

```
version: '3'

services:
  web:
    build:
      context: ./docker
      dockerfile: web.docker
    volumes:
      - ./:/var/www
    ports:
      - "127.0.0.1:80:80"
    links:
      - app

  app:
    build:
      context: ./docker
      dockerfile: app.docker
    volumes:
      - ./:/var/www
    links:
      - database
      - redis
      - "elasticsearch:elasticsearch"

  redis:
    image: redis

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.1
    environment:
      - discovery.type=single-node

  kibana:
    image: docker.elastic.co/kibana/kibana:7.6.1
    links:
      - "elasticsearch:elasticsearch"
    ports:
      - "127.0.0.1:5601:5601"


  database:
    image: mysql:5.7
    env_file:
      - ./docker.env

  mailhog:
    image: mailhog/mailhog
    ports:
      - 127.0.0.1:8025:8025
```

And below is the modified version for arm64v8 (raspberry pi 3 and raspberry pi 4 compatible):

```
version: '3'

services:
  web:
    build:
      context: ./docker
      dockerfile: web.docker
    volumes:
      - ./:/var/www
    ports:
      - "127.0.0.1:80:80"
    links:
      - app

  app:
    build:
      context: ./docker
      dockerfile: app.docker
    volumes:
      - ./:/var/www
    links:
      - database
      - redis
      - "elasticsearch:elasticsearch"

  redis:
    image: redis

  elasticsearch:
    image: arm64v8/elasticsearch:7.8.1
    environment:
      - discovery.type=single-node

        #kibana:
        #    image: docker.elastic.co/kibana/kibana:7.6.1
        #    links:
        #      - "elasticsearch:elasticsearch"
        #    ports:
        #      - "127.0.0.1:5601:5601"


  database:
    image: arm64v8/mariadb
    env_file:
      - ./docker.env

        #  mailhog:
        #    image: mailhog/mailhog
        #    ports:
        #      - 127.0.0.1:8025:8025

```

The `web` and `app` containers are compiled natively when the docker image is built - which is helped by the fact they are based on Debian containers which have good arm64 support.

I had to replace the elasticsearch and mariadb image with a ported image from a user called arm64v8, but is basically a fork of the official repository, but compiled on an arm64v8 processor.

Unfortunately, I couldn't find an arm64v8 compatible image that was prebuilt for mailhog, nor for kibana - but when I have some time I'll put some together and link them to this article.
