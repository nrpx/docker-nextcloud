ARG NC_VERSION=31-fpm-alpine

FROM nextcloud:${NC_VERSION}

ARG SMBCLIENT_VERSION=1.1.2

COPY opcache-recommended.ini /usr/local/etc/php/conf.d/opcache-recommended.ini

RUN set -ex; \
    \
    apk upgrade --update-cache --available \
    ; \
    \
    apk add --no-cache \
        openssl \
        ffmpeg \
        procps \
        samba-client \
    ; \
    \
    rm -rf /var/cache/apk/*

RUN set -ex; \
    \
    apk add --no-cache --virtual .build-deps \
        $PHPIZE_DEPS \
        samba-dev \
        bzip2-dev \
    ; \
    \
    docker-php-ext-install \
        bz2 \
    ; \
    \
    pecl install "https://pecl.php.net/get/smbclient-${SMBCLIENT_VERSION}.tgz"; \
    docker-php-ext-enable smbclient; \
    \
    runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )"; \
    apk add --virtual .nextcloud-phpext-rundeps $runDeps; \
    apk del .build-deps
