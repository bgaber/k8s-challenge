FROM php:7.4-apache

RUN docker-php-ext-install mysqli pdo pdo_mysql && docker-php-ext-enable pdo_mysql

COPY ./learning-app-ecommerce/ /var/www/html/

EXPOSE 80
