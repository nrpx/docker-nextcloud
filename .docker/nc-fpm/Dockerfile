ARG NC_VERSION=32-fpm-alpine

FROM nextcloud:${NC_VERSION}

ARG SMBCLIENT_VERSION=1.1.2
ARG SMBCLIENT_SHA256=042465bdde187eb32d4233402f88368426d76c84ab1d076701b6592d6b5fa5d8

COPY opcache-recommended.ini /usr/local/etc/php/conf.d/opcache-recommended.ini

# Download the pinned SMB release from upstream GitHub, avoiding the PECL host.
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
    curl --fail --location --retry 5 --retry-all-errors \
        --connect-timeout 15 --max-time 120 \
        "https://codeload.github.com/eduardok/libsmbclient-php/tar.gz/refs/tags/${SMBCLIENT_VERSION}" \
        --output /tmp/smbclient.tar.gz; \
    echo "${SMBCLIENT_SHA256}  /tmp/smbclient.tar.gz" | sha256sum -c -; \
    docker-php-source extract; \
    mkdir -p /usr/src/php/ext/smbclient; \
    tar -xzf /tmp/smbclient.tar.gz --strip-components=1 -C /usr/src/php/ext/smbclient; \
    docker-php-ext-install -j "$(nproc)" bz2 smbclient; \
    docker-php-source delete; \
    rm /tmp/smbclient.tar.gz; \
    \
    runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )"; \
    apk add --virtual .nextcloud-phpext-rundeps $runDeps; \
    apk del .build-deps; \
    php --ri smbclient; \
    php -r 'exit(extension_loaded("smbclient") && extension_loaded("bz2") && function_exists("smbclient_state_new") ? 0 : 1);'; \
    smbclient --version
