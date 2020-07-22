## TUTORIAL: How To Set Up Laravel, Nginx, and MySQL with Docker Compose

<p align="center">
<a href = "#">Nginx</a>
<a href = "#">MySQL</a>
<a href = "#">PHP</a>
<a href = "#">Docker</a>
<a href = "#">PHP Frameworks</a>
<a href = "#">Ubuntu</a>
<a href = "#">Laravel</a>
</p>

### Step 1 — Downloading Laravel and Installing Dependencies
First, check that you are in your home directory and clone the latest Laravel release to a directory called laravel-app
```
cd ~
git clone https://github.com/laravel/laravel.git laravel-app 
``` 

Move into the laravel-app directory:

```cd ~/laravel-app``` 

Next, use Docker’s composer image to mount the directories that you will need for your Laravel project and avoid the overhead of installing Composer globally:

```docker run --rm -v $(pwd):/app composer install``` 

Using the -v and --rm flags with docker run creates an ephemeral container that will be bind-mounted to your current directory before being removed. This will copy the contents of your ~/laravel-app directory to the container and also ensure that the vendor folder Composer creates inside the container is copied to your current directory.

As a final step, set permissions on the project directory so that it is owned by your non-root user:

```sudo chown -R $USER:$USER ~/laravel-app``` 

This will be important when you write the Dockerfile for your application image in Step 4, as it will allow you to work with your application code and run processes in your container as a non-root user.

With your application code in place, you can move on to defining your services with Docker Compose.

### Step 2 — Creating the Docker Compose File

```
version: '3'
services:

  #PHP Service
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: digitalocean.com/php
    container_name: app
    restart: unless-stopped
    tty: true
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    working_dir: /var/www
    volumes:
      - ./:/var/www
      - ./php/local.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      - app-network

  #Nginx Service
  webserver:
    image: nginx:alpine
    container_name: webserver
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./:/var/www
      - ./nginx/conf.d/:/etc/nginx/conf.d/
    networks:
      - app-network

  #MySQL Service
  db:
    image: mysql:5.7.22
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - mysqldata:/var/lib/mysql/
      - ./mysql/my.cnf:/etc/mysql/my.cnf
    networks:
      - app-network

#Docker Networks
networks:
  app-network:
    driver: bridge
#Volumes
volumes:
  mysqldata:
    driver: local
```

### Step 4 — Creating the Dockerfile

```
FROM php:7.2-fpm

# Copy composer.lock and composer.json
COPY composer.lock composer.json /var/www/

# Set working directory
WORKDIR /var/www

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    jpegoptim optipng pngquant gifsicle \
    vim \
    unzip \
    git \
    curl

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install extensions
RUN docker-php-ext-install pdo_mysql mbstring zip exif pcntl
RUN docker-php-ext-configure gd --with-gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ --with-png-dir=/usr/include/
RUN docker-php-ext-install gd

# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Add user for laravel application
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www

# Copy existing application directory contents
COPY . /var/www

# Copy existing application directory permissions
COPY --chown=www:www . /var/www

# Change current user to www
USER www

# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
```

### Step 5 — Configuring PHP

```
docker-compose exec app php artisan key:generate
docker-compose exec app php artisan config:cache
docker-compose exec app php artisan migrate
```

