# Testing the blackfire SDK on a SF 5.0.8 project

This is a simple website starter project on Symfony 5.0.8.

It runs on a PHP 7.4.6 alpine fpm container :

```shell script
FROM php:7.4.6-fpm-alpine

ENV APCU_VERSION 5.1.18
ENV MCRYPT_VERSION 1.0.3

COPY my-config.ini /usr/local/etc/php/conf.d/.
COPY opcache.ini /usr/local/etc/php/conf.d/.

RUN apk --no-cache update \
    && apk --no-cache upgrade

RUN apk add --no-cache \
        $PHPIZE_DEPS \
        libzip-dev \
        freetype-dev \
        libjpeg-turbo-dev \
        libmcrypt-dev \
        zlib-dev \
        libpng-dev \
        libwebp-dev \
        icu-dev \
        g++ \
        exiftool \
        git \
        su-exec \
        libxslt-dev \
        libgcrypt-dev

RUN pecl install apcu-${APCU_VERSION} \
    && docker-php-ext-enable apcu \
    && pecl install mcrypt-${MCRYPT_VERSION} \
    && docker-php-ext-enable mcrypt
    
#Configure, install and enable all php packages
RUN docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr/include/ --with-webp=/usr/include/ \
    && docker-php-ext-configure opcache --enable-opcache \
    && docker-php-ext-configure intl \
    && docker-php-ext-configure exif

RUN docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-install zip \
    && docker-php-ext-install intl \
    && docker-php-ext-install iconv \
    && docker-php-ext-install exif \
    && docker-php-ext-install opcache \
    && docker-php-ext-install xsl

#Install composer
RUN wget https://raw.githubusercontent.com/composer/getcomposer.org/4d7f8d40f9788de07c7f7b8946f340bf89535453/web/installer -O - -q | php -- --install-dir=/usr/local/bin --filename=composer

#Remove cached files
RUN docker-php-source delete \
    && pecl clear-cache \
    && rm -rf \
        /usr/include/php \
        /usr/lib/php/build \
        /tmp/* \
        /var/lib/apt/lists/* \
        /root/.composer

WORKDIR /var/www/symfony

RUN PATH=$PATH:/usr/src/apps/vendor/bin:bin
```

When running it with just the default composer packages it runs fine and displays to Symfony starter page in the browser.

However when running `composer require blackfire/php-sdk` (from inside the container) the following error appears :

```shell script
Using version ^1.22 for blackfire/php-sdk
./composer.json has been updated
Loading composer repositories with package information
Updating dependencies (including require-dev)
Restricting packages listed in "symfony/symfony" to "5.0.*"

Prefetching 2 packages 🎶 💨
  - Downloading (100%)

Package operations: 2 installs, 0 updates, 0 removals
  - Installing composer/ca-bundle (1.2.7): Loading from cache
  - Installing blackfire/php-sdk (v1.22.0): Loading from cache
Writing lock file
Generating autoload files
ocramius/package-versions: Generating version class...
ocramius/package-versions: ...done generating version class
74 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
Symfony operations: 1 recipe (f9c976ba9e09a5920a7f0a109b08496f)
  - Configuring composer/ca-bundle (>=1.2.7): From auto-generated recipe
Executing script cache:clear [KO]
 [KO]
Script cache:clear returned with error code 255
!!  Symfony\Component\ErrorHandler\Error\UndefinedMethodError {#50
!!    #message: "Attempted to call an undefined method named "getName" of class "Composer\CaBundle\CaBundle"."
!!    #code: 0
!!    #file: "./vendor/symfony/http-kernel/Kernel.php"
!!    #line: 370
!!    trace: {
!!      ./vendor/symfony/http-kernel/Kernel.php:370 { …}
!!      ./vendor/symfony/http-kernel/Kernel.php:123 { …}
!!      ./vendor/symfony/framework-bundle/Console/Application.php:168 { …}
!!      ./vendor/symfony/framework-bundle/Console/Application.php:74 { …}
!!      ./vendor/symfony/console/Application.php:140 { …}
!!      ./bin/console:42 {
!!        › $application = new Application($kernel);
!!        › $application->run($input);
!!        › 
!!        arguments: {
!!          $input: Symfony\Component\Console\Input\ArgvInput {#6 …}
!!        }
!!      }
!!    }
!!  }
!!  2020-05-25T14:52:00+02:00 [critical] Uncaught Error: Call to undefined method Composer\CaBundle\CaBundle::getName()
!!  
Script @auto-scripts was called via post-update-cmd

Installation failed, reverting ./composer.json to its original content.
```

The composer version is 1.10.6