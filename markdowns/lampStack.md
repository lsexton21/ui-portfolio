## Example of LAMP stack for local development

This is one example of a docker container that I have built.  I chose this one to share because it builds the newest versions of everything and includes a custom dockerfile build and a custom file to configure apache's vhost settings for routing purposes.  I used this to locally develop a LAMP stack application.

---

#### This is the dockerfile to create the custom php-8.2-apache server with custom apache vhost configs
```
FROM php:8.2-apache

# Installing dependencies for building the PHP modules
RUN apt update && \
    apt install -y zip libzip-dev libpng-dev libicu-dev libxml2-dev

COPY apache-vhost.conf /etc/apache2/sites-available/apache-vhost.conf
RUN a2enmod deflate rewrite proxy proxy_fcgi && a2dissite 000-default && a2ensite apache-vhost && service apache2 restart

# Installing additional PHP modules
RUN docker-php-ext-install mysqli pdo pdo_mysql gd zip intl xml

# Installing composer
COPY --from=composer /usr/bin/composer /usr/bin/composer

ENV COMPOSER_ALLOW_SUPERUSER=1

COPY www /var/www/html

WORKDIR /var/www/html

# # Installing dependencies for building the PHP modules
RUN composer install
RUN composer dump-autoload

# Cleaning APT cache
RUN apt clean
```
---
#### Adding the virtual host settings for the apache web server
```
<VirtualHost *:80>
    DocumentRoot /var/www/html/public
    <Directory /var/www/html/public>
        # Sets the default file to load
        DirectoryIndex index.php

        # Prevents files from being viewed as a directory tree
        Options Indexes FollowSymLinks

        # Enables .htaccess files if present
        AllowOverride All

        Require all granted
    </Directory>
    
    # Send Apache logs to stdout and stderr
    CustomLog /proc/self/fd/1 common
    ErrorLog /proc/self/fd/2
</VirtualHost>
```

---
#### Adding the docker compose file that will add mariaDB and phpMyAdmin to the php-apache build, map my volumes for real-time editing
```
version: "2"
services:

  # PHP-Apache Service
  php:
    container_name: lamp-stack
    image: custom-php-8.2-apache
    build: '.'
    ports:
      - "80:80"
    volumes:
      - ./www/public:/var/www/html/public
      - ./www/src:/var/www/html/src
      - ./www/cli.php:/var/www/html/cli.php
      - ./www/composer.json:/var/www/html/composer.json
      - ./www/database.sql:/var/www/html/database.sql

    depends_on:
      - mariadb

  # MariaDB Service
  mariadb:
    container_name: mariadb
    image: mariadb:10.11
    environment:
      MYSQL_ROOT_PASSWORD: NOT TELLING
    volumes:
      - mysqldata:/var/lib/mysql

  # phpMyAdmin Service
  phpmyadmin:
    container_name: phpmyadmin
    image: phpmyadmin/phpmyadmin:latest
    ports:
      - 8080:80
    environment:
      PMA_HOST: mariadb
    depends_on:
      - mariadb

volumes:
  mysqldata:
```