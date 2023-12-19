FROM php:8.3.0-alpine

# Add docker-php-extension-installer script
ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

# Install dependencies
RUN apk add --no-cache \
    bash \
    curl \
    freetype \
    g++ \
    gcc \
    git \
    icu-dev \
    icu-libs \
    libc-dev \
    libpq-dev \
    libpng-dev \
    freetype-dev \
    libjpeg-turbo-dev \
    sqlite \
    sqlite-dev \
    libzip-dev \
    make \
    mysql-client \
    nodejs \
    npm \
    oniguruma-dev \
    yarn \
    openssh-client \
    postgresql-libs \
    rsync \
    zlib-dev \
    gcompat \
    libstdc++ \
    nss \
    harfbuzz \
    ca-certificates \
    ttf-freefont \
    fontconfig \
    dbus \
    chromium \
    chromium-chromedriver \
    ruby  \
    ruby-dev  \
    ruby-bigdecimal  \
    build-base

# Install php extensions
RUN chmod +x /usr/local/bin/install-php-extensions && \
    install-php-extensions \
    @composer \
    redis-stable \
    imagick-stable \
    xdebug-stable \
    bcmath \
    calendar \
    exif \
    gd \
    intl \
    pdo_mysql \
    pdo_pgsql \
    pcntl \
    soap \
    zip \
    sockets \
    gettext \
    shmop \
    sysvmsg \
    swoole

# Add local and global vendor bin to PATH.
ENV PATH ./vendor/bin:/composer/vendor/bin:/root/.composer/vendor/bin:/usr/local/bin:$PATH

# Install PHP_CodeSniffer
RUN composer global require "squizlabs/php_codesniffer=*"

# Setup working directory
WORKDIR /var/www
