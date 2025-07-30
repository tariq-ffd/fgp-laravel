# Stage 1: Builder - for installing Composer and Node dependencies
FROM php:8.3-cli-alpine AS builder

# Arguments for user and group IDs
ARG UID=1000
ARG GID=1000

# Install system dependencies needed for Composer and Node (only in this build stage)
RUN apk add --no-cache \
    git \
    curl \
    nodejs \
    npm \
    # Other build-time dependencies like python3 for some npm packages if needed
    && rm -rf /var/cache/apk/*

# Install Composer
COPY --from=composer:latest /usr/local/bin/composer /usr/local/bin/composer

# Set working directory
WORKDIR /app

# Copy composer files first to leverage Docker cache
COPY composer.json composer.lock ./
RUN composer install --no-dev --prefer-dist --optimize-autoloader --no-interaction

# Copy Node/NPM files
# Use package-lock.json if you have it, or yarn.lock if using yarn
COPY package.json package-lock.json ./ 
RUN npm install

# Copy application code
COPY . .

# Build frontend assets (this can be done here or in a separate frontend build stage)
# This uses the installed node_modules
RUN npm run build


# Stage 2: Production - the final, lean PHP-FPM image
# Using Alpine for a smaller final image
FROM php:8.3-fpm-alpine 

# Arguments for user and group IDs
ARG UID=1000
ARG GID=1000

# Install runtime system dependencies for Laravel
# Using --no-cache for smaller image, --update for latest packages
RUN apk update --no-cache && apk add --no-cache \
    libpng-dev \
    libjpeg-turbo-dev \
    libwebp-dev \
    libzip-dev \
    libonig-dev \
    libxml2-dev \
    libpq-dev \
    libintl \
    icu-dev \
    supervisor \
    # Add other runtime libs needed by extensions, e.g., mariadb-client-libs if using mariadb
    && rm -rf /var/cache/apk/*

# Install PHP extensions
# The '--with-xxx-dir' flags are for gd extension with specific libraries
RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install -j$(nproc) \
        pdo_mysql \ 
        gd \
        opcache \
        exif \
        pcntl \
        bcmath \
        ctype \
        fileinfo \
        mbstring \
        tokenizer \
        xml \
        zip \
        intl \
    && rm -rf /tmp/pear \
    && rm -rf /usr/share/doc

# Install Redis extension using pecl (needs to be done in the final stage)
RUN pecl install redis && docker-php-ext-enable redis

# Set working directory for the application
WORKDIR /var/www/html

# Create a non-root user and group, ensuring IDs match host or are configurable
RUN addgroup -g ${GID} www-data \
    && adduser -u ${UID} -G www-data -s /bin/sh -D www-data

# Copy built application from the builder stage
COPY --from=builder /app /var/www/html

# Set appropriate permissions for Laravel
# Ensure www-data owns the entire app directory and specific Laravel writable directories
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 775 /var/www/html/storage \
    && chmod -R 775 /var/www/html/bootstrap/cache

# Copy custom PHP configuration files
# Place these in docker/php/ in your project
COPY docker/php/php.ini /usr/local/etc/php/conf.d/custom.ini
# If you have a www.conf for FPM pool settings
# COPY docker/php/www.conf /usr/local/etc/php-fpm.d/www.conf

USER www-data

# Command to run Supervisor, which in turn will manage php-fpm and other processes
# You'll need a supervisor config file copied into the image
# Example: ADD docker/php/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
# CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf", "-n"]

# If you just want to run php-fpm for development and don't need supervisor yet
CMD ["php-fpm"]

EXPOSE 9000