FROM php:8.1-apache

RUN apt-get update && apt-get install -y \
    git libzip-dev zip unzip npm \
    certbot python3-certbot-apache

RUN docker-php-ext-install pdo pdo_mysql zip
RUN a2enmod rewrite

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Copy virtual host config

COPY vhosts.conf /etc/apache2/sites-available/
RUN a2dissite 000-default.conf
RUN a2ensite vhosts.conf

RUN service apache2 restart