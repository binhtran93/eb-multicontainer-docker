FROM php:7.2-fpm
RUN apt-get update && apt-get install -y \
    libzip-dev \
    zip \
    unzip
RUN docker-php-ext-configure zip --with-libzip
RUN docker-php-ext-install pdo_mysql zip

RUN chown -R www-data:www-data /var/www

#CMD ["php-fpm"]