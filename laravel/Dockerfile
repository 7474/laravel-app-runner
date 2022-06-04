# Install dependencies only when needed
FROM node:16-alpine AS node-deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

# Rebuild the source code only when needed
FROM node:16-alpine AS node-builder
WORKDIR /app
COPY . .
COPY --from=node-deps /app/node_modules ./node_modules
RUN yarn production

# For app-runner
# https://learn2torials.com/a/laravel8-production-docker-image
# https://gitlab.com/learn2torials/laravel8-docker
FROM php:8.1-fpm AS laravel-app-runner

# Set working directory
WORKDIR /var/www

# Add docker php ext repo
# https://github.com/mlocati/docker-php-extension-installer/
ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

# Install php extensions
RUN chmod +x /usr/local/bin/install-php-extensions && sync && \
    install-php-extensions mbstring pdo_mysql zip exif pcntl gd memcached

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    jpegoptim optipng pngquant gifsicle \
    unzip \
    git \
    curl \
    lua-zlib-dev \
    libmemcached-dev \
    nginx

# Install supervisor
RUN apt-get install -y supervisor

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Add user for laravel application
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www

# Copy code to /var/www
COPY --chown=www:www-data . /var/www

# add root to www group
RUN chmod -R ug+w /var/www/storage

# Deployment steps
# https://laravel.com/docs/9.x/deployment
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
RUN composer install --optimize-autoloader --no-dev
# XXX 成果物だけコピー
RUN rm /usr/local/bin/composer

# Configure nginx/php/supervisor configs
# php-fpm のアクセスログはノイジーなので出力しない
RUN sed -ri -e 's!access.log = /proc/self/fd/2!access.log = /proc/self/fd/1!g' /usr/local/etc/php-fpm.d/docker.conf
COPY docker/cloud/supervisord.conf /etc/supervisord.conf
COPY docker/cloud/php.ini /usr/local/etc/php/conf.d/app.ini
COPY docker/cloud/nginx.conf /etc/nginx/sites-enabled/default
# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log
RUN ln -sf /dev/stdout /var/log/nginx/php-access.log \
    && ln -sf /dev/stderr /var/log/nginx/php-error.log
RUN ln -sf /dev/stdout /var/log/nginx/schedule.log \
    && ln -sf /dev/stdout /var/log/nginx/notification.log \
    && ln -sf /dev/stdout /var/log/nginx/worker.log

COPY docker/cloud/start-container /usr/local/bin/start-container
RUN chmod +x /usr/local/bin/start-container

COPY --from=node-builder /app/public ./public

EXPOSE 80
ENTRYPOINT ["start-container"]
