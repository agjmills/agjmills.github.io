---
layout: post
title: Running Laravel inside Docker for local development
---

A simple tutorial project explaining fundamentals of docker, in the context of a Laravel application

This is written from the perspective of an Ubuntu Linux user (me), so YMMV if you're using Mac/Windows/Other Linux 
Distributions

You can download all of the code and steps for this tutorial here: https://github.com/agjmills/laravel-docker-tutorial

## Steps

1. Installing docker and docker-compose
2. How a docker-compose.yml file works
3. How docker images are built
4. Mounting the application into the container
5. Configuring container environments
6. Day to day development with Docker & Laravel

## Contributing

Please submit a pull request or add an issue if I've missed something or you have an idea. I whole-heartedly welcome them!

I'll do my best to add bits and pieces as I come across them, too.app.docker


# Installing docker, and docker-compose

## Installation
### Linux
Update apt cache, remove old versions of docker, install and enable docker to start on boot
```
sudo apt-get update
sudo apt-get remove docker docker-engine docker.io
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker
```
Add your user to the `docker` group:
```
sudo gpasswd -a alex docker
```
Install docker-compose:
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

### Mac
Follow this guide: https://docs.docker.com/docker-for-mac/install/

Docker Desktop for Mac and Docker Toolbox already include Compose along with other Docker apps, so Mac users do not need to install Compose separately.

### Windows
Follow this guide: https://docs.docker.com/docker-for-windows/install/

Docker Desktop for Windows and Docker Toolbox already include Compose along with other Docker apps, so most Windows users do not need to install Compose separately.

## Running docker-compose
If you run `docker-compose up -d` in the root of this project, it should build the images as necessary, and start the containers.

You can see them running via `docker ps`

You can stop and remove the containers by running `docker-compose down` (!) This will REMOVE containers, so may delete databases.

You can stop and NOT remove containers by running `docker-compose stop`.


# How `docker-compose.yml` works

A `docker-compose.yml` file can be seen as a YAML description of your applications stack.

## Connecting to each service

So, with the `docker-compose.yml` in this project, you can see several "services":

* web (nginx)
* app (php-fpm)
* redis
* elasticsearch
* kibana
* database (mysql)
* mailhog

Each of these runs in a separate container, and is addressable via the name of the service from within another container.

So, if you wanted to connect to the container called `elasticsearch` from the container called `app`, you might do something 
within the app container like this: `curl http://elasticsearch:9200`.

You'll notice, that the elasticsearch service doesn't have a `ports` attribute - this means the port inside the container
isn't bound to a port on the host system. 

But, if you look at the `web` service, that has a port definition of `127.0.0.1:80:80`, which means that `127.0.0.1`, port `80`, is bound to port `80` inside the container.
So we should be able to open our web browser and navigate to `http://127.0.0.1` and we will see the Laravel default screen.

I have deliberately included `.env` witin the application directory so that you can see how to configure Laravel to connect
to the various services.

## Service links

Some of the services have a linked attribute - these are basically showing the dependencies between the services,
and it gives `docker-compose` an idea of which order to start certain services in. 

Going from the top of the file, we can see that `web` is dependent on `app`, `app` is dependent on `database`, `redis` and `elasticsearch`. So 
they'll be started in roughly that order.   

# How docker images are built
You can see that for the services `web` and `app` they have a `build` attribute.

This shows `docker-compose` that if an image does not exist already, how to build one.

```
  app:
    build:
      context: ./docker
      dockerfile: app.docker
``` 

The app service is build from the `app.docker` file, within the `docker/` directory.

## Dockerfiles

Below is `docker/app.docker`. It describes how to build the image for the `app` service. 
```
FROM php:7.4-fpm-alpine

RUN apk update && apk add mysql-client \
        freetype \
        libpng \
        libjpeg-turbo \
        freetype-dev \
        libpng-dev \
        libjpeg-turbo-dev \
        $PHPIZE_DEPS \
     && docker-php-ext-configure gd \
        --with-freetype \
        --with-jpeg \
    && docker-php-ext-install pdo_mysql mysqli gd exif

ARG WITH_XDEBUG=false
RUN if [ $WITH_XDEBUG = "true" ] ; then \
        pecl install xdebug; \
        docker-php-ext-enable xdebug; \
        echo "error_reporting = E_ALL" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini; \
        echo "display_startup_errors = On" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini; \
        echo "display_errors = On" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini; \
        echo "xdebug.remote_enable=1" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini; \
    fi;

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

WORKDIR /var/www
```

It takes an existing image, `php:7.4-fpm-alpine` (you can find more on https://hub.docker.com/), and runs some commands. 

This particular example uses Alpine Linux (which is a very lightweight linux distribution), and installs some dependencies
for PHP, such as pdo_mysql, php-mysqli, php-gd and php-exif. 

It also installs some packages from a variable, `$PHPIZE_DEPS`. These are all of the packages required to run the `phpize`
command required to install certain PHP extensions such as XDebug.

There is also conditional section towards the bottom of the file. If the `WITH_XDEBUG` argument is defined as true, 
install, configure and enable xdebug package.

Finally, it defines the workdir as `/var/www`  

# Mounting the application into the container

You will see in certain services within `docker-compose.yml`, that they have a `volumes` attribute.

This defines what files or folders will be mounted inside the container. They are _mounted_ and not _copied_, so 
any changes you make outside of the container will be reflected inside the container, too.

Looking at the `app` service:

```
    volumes:
      - ./application:/var/www
``` 

This mounts the application directory (which contains Laravel) to `/var/www` inside the container.

# Configuring Container Environments
It's a good idea to build generic docker images, and then configure them within the context of 
your application. As an example, you might build a generic `elasticsearch` container, and allow the user of that
container to set the maximum memory limit, how many nodes are in the cluster, and so on.

This allows you to build fewer docker images, and therefore follows the DRY principle.

## Configuring using the `environment` attribute
If you look at the `elasticsearch` service, it has an `environment` attribute.

This sets a setting within the elasticsearch configuration. In this specific example, it stop elasticsearch from looking 
for other nodes to cluster with. 

## Configuring using an external file

If you look at the `database` service, it has an attribute named `env_file`. This works in the same way as the `environment`
attribute, it just configures the service inside the container in a certain way.

In this specific example: 

* It creates a database called `laravel` if it doesn't already exist
* It sets the MySQL root password to `secret`
* It sets the database port to `3306`
* It sets the database hostname to `database`

Using these settings, we should be able to connect to MySQL like this:

`docker exec -ti laravel-docker-tutorial_app_1 mysql -uroot -psecret -hdatabase laravel`

This runs the mysql command inside the `app` service.

## Notes

Environment variables are different for every service, but they are generally documented somewhere, it's best to look at 
https://hub.docker.com

# Day to day development with Docker

This section contains a few things I've picked up after having developed Laravel projects using docker-compose on a day-to-day basis. 

## Learn the difference between `docker-compose stop` and `docker-compose down`
* `docker-compose down`: Stop and remove containers, networks, images, and volumes
* `docker-compose stop`: Stop services

Use `down` if you want to completly trash everything and start again, but use `stop` if you will be coming back to it at 
some point in the future.

## Go inside the container and have a look around
You can go inside any container by executing the following:

`docker exec -ti NAME_OF_CONTAINER bash` or `docker exec -ti NAME_OF_CONTAINER sh`  (if bash isnt installed)

## Use Laravel generators _outside_ of the container
If you run something like `php artisan make:migration add_users_table`, inside the container, it is run as `root`, which 
means that root then owns the file, and you'll have to `sudo chown -R your_user:your_user ./application/database/migrations`
to change the owner.

If you run it outside of the container, it'll create it with you as the owner instead.

## Execute Laravel & Artisan commands _inside_ the container
If you need to run the migrations, you _have_ to do it inside the container:

`docker exec -ti laravel-docker-tutorial_app_1 php artisan migrate`

Otherwise the application won't be able to resolve the hostname `database` for the MySQL server.

## Trash everything and rebuild from scratch
Sometimes, you break everything, or you just _really_ have to rebuild everything from scratch as it's not working properly.
Here are a few useful commands:

* `docker-compose up -d --force-recreate` Recreate containers even if their configuration and image haven't changed.
* `docker-compose up -d --force-recreate` Build images before starting containers.
    
## XDebug
When built, the app container image within this repository will have xdebug installed (if `WITH_XDEBUG=true` is defined within 
the `app.build.args` attribute inside `docker-compose.yml`).

By default, this will attempt to connect to `10.0.1.41` on port `9000`. You can change this by editing the `docker.env` file. Change the
IP address defined here to your local IP address, and [setup your IDE to accept connections on port 9000](https://dev.to/oranges13/phpstorm-xdebug-alpine-on-docker-13ff#setting-up-phpstorm). 
