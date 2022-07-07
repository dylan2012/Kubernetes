FROM php:7.4.29-fpm-alpine3.15
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/
#ADD install-php-extensions /usr/local/bin/
RUN apk update && apk add wget curl git vim
RUN docker-php-ext-install pdo_mysql
RUN docker-php-ext-install bcmath
RUN docker-php-ext-install mysqli
RUN docker-php-ext-install sockets

RUN chmod uga+x /usr/local/bin/install-php-extensions && sync
RUN install-php-extensions redis zip pcov  pcntl gd exif xlswriter memcached xmlrpc opcache mongodb
#RUN install-php-extensions swoole