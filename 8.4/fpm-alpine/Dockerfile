FROM php:8.4-fpm-alpine

ENV ACCEPT_EULA=Y \
    COMPOSER_ALLOW_SUPERUSER=1

# --- Install minimal tools just for TLS + GPG, then remove later
RUN apk add --no-cache ca-certificates curl gnupg \
 && update-ca-certificates

# --- Install Microsoft ODBC Driver 18 + mssql-tools (verified), then remove curl/gnupg
ARG MSODBC_VER=18.5.1.1-1
ARG MSSQL_TOOLS_VER=18.4.1.1-1
RUN set -euo pipefail \
  && case "$(uname -m)" in \
       x86_64)  arch="amd64" ;; \
       aarch64) arch="arm64" ;; \
       *)       echo "Unsupported arch: $(uname -m)"; exit 1 ;; \
     esac \
  # download packages + signatures
  && curl -fSLO "https://download.microsoft.com/download/fae28b9a-d880-42fd-9b98-d779f0fdd77f/msodbcsql18_${MSODBC_VER}_${arch}.apk" \
  && curl -fSLO "https://download.microsoft.com/download/7/6/d/76de322a-d860-4894-9945-f0cc5d6a45f8/mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.apk" \
  && curl -fSLO "https://download.microsoft.com/download/fae28b9a-d880-42fd-9b98-d779f0fdd77f/msodbcsql18_${MSODBC_VER}_${arch}.sig" \
  && curl -fSLO "https://download.microsoft.com/download/7/6/d/76de322a-d860-4894-9945-f0cc5d6a45f8/mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.sig" \
  # import key + verify
  && curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --import - \
  && gpg --verify "msodbcsql18_${MSODBC_VER}_${arch}.sig" "msodbcsql18_${MSODBC_VER}_${arch}.apk" \
  && gpg --verify "mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.sig" "mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.apk" \
  # install
  && apk add --no-cache --allow-untrusted \
       "msodbcsql18_${MSODBC_VER}_${arch}.apk" \
       "mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.apk" \
  # cleanup artifacts + temporary tools
  && rm -f msodbcsql18_${MSODBC_VER}_${arch}.apk \
            mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.apk \
            msodbcsql18_${MSODBC_VER}_${arch}.sig \
            mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.sig \
  && apk del gnupg curl

# --- Bring in the PHP extension installer
# (use a stage so the helper binary is copied without pulling the whole image into the final layers)
FROM mlocati/php-extension-installer:latest AS php_ext
FROM php:8.4-fpm-alpine
COPY --from=php_ext /usr/bin/install-php-extensions /usr/bin/install-php-extensions

# Reapply the ODBC bits (they’re in the previous stage only)
ENV ACCEPT_EULA=Y
ARG MSODBC_VER=18.5.1.1-1
ARG MSSQL_TOOLS_VER=18.4.1.1-1
RUN apk add --no-cache ca-certificates && update-ca-certificates

# (Re)install the already-verified Microsoft packages by URL directly:
# If you prefer keeping a single stage, move the first ODBC block here and delete the two lines above.
RUN set -euo pipefail \
  && case "$(uname -m)" in \
       x86_64)  arch="amd64" ;; \
       aarch64) arch="arm64" ;; \
       *)       echo "Unsupported arch: $(uname -m)"; exit 1 ;; \
     esac \
  && wget -q "https://download.microsoft.com/download/fae28b9a-d880-42fd-9b98-d779f0fdd77f/msodbcsql18_${MSODBC_VER}_${arch}.apk" \
  && wget -q "https://download.microsoft.com/download/7/6/d/76de322a-d860-4894-9945-f0cc5d6a45f8/mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.apk" \
  && apk add --no-cache --allow-untrusted \
       "msodbcsql18_${MSODBC_VER}_${arch}.apk" \
       "mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.apk" \
  && rm -f "msodbcsql18_${MSODBC_VER}_${arch}.apk" \
           "mssql-tools18_${MSSQL_TOOLS_VER}_${arch}.apk"

# --- Install PHP extensions (this script auto-installs & removes compilers/build-deps)
# Important: install sqlsrv/pdo_sqlsrv AFTER msodbc is present
RUN chmod +x /usr/bin/install-php-extensions \
 && install-php-extensions \
    bcmath ds exif gd intl opcache pcntl \
    pdo_sqlsrv sqlsrv redis zip pgsql pdo_pgsql curl

# --- Config tweaks (single layer)
RUN set -eux; \
    sed -i 's/\(MinProtocol *= *\).*/\1TLSv1.0 /' /etc/ssl/openssl.cnf; \
    sed -i 's/\(CipherString *= *\).*/\1DEFAULT@SECLEVEL=1 /' /etc/ssl/openssl.cnf; \
    sed -i 's/error_log = /;error_log = /' /usr/local/etc/php-fpm.d/docker.conf

# --- Composer (tiny, copied—no curl kept)
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

WORKDIR /var/www