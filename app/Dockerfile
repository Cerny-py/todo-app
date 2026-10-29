FROM php:8.2-apache
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
RUN docker-php-ext-install pdo pdo_mysql
COPY index.php /var/www/html/
EXPOSE 80
