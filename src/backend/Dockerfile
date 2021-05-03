FROM php:8.0-fpm-alpine

# Install extensions
RUN docker-php-ext-install pdo_mysql bcmath opcache

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# Configura PHP
COPY .docker/php.ini $PHP_INI_DIR/conf.d/opcache.ini

# Usa la configurazione di sviluppo predefinita
RUN mv $PHP_INI_DIR/php.ini-development $PHP_INI_DIR/php.ini
