FROM php:8.1-fpm-buster

ENV ACCEPT_EULA=Y

# Install prerequisites required for tools and extensions installed later on.
RUN apt-get update \
    && apt-get install -y apt-transport-https gnupg2 libpng-dev libzip-dev unzip \
    && rm -rf /var/lib/apt/lists/*

# Install prerequisites for the sqlsrv and pdo_sqlsrv PHP extensions.
RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
    && curl https://packages.microsoft.com/config/debian/10/prod.list > /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update \
    && apt-get install -y msodbcsql17 mssql-tools unixodbc-dev \
    && rm -rf /var/lib/apt/lists/*

# Retrieve the script used to install PHP extensions from the source container.
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/bin/install-php-extensions

# Install required PHP extensions and all their prerequisites available via apt.
RUN chmod uga+x /usr/bin/install-php-extensions \
    && sync \
    && install-php-extensions bcmath ds exif gd intl opcache pcntl pdo_sqlsrv redis sqlsrv zip

# Set Min Protocol OpenSSL TLS1.0
RUN sed -i "s/\(MinProtocol *= *\).*/\1TLSv1.0 /" "/etc/ssl/openssl.cnf"
RUN sed -i "s/\(CipherString *= *\).*/\1DEFAULT@SECLEVEL=1 /" "/etc/ssl/openssl.cnf"

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# Setting the work directory.
WORKDIR /var/www
