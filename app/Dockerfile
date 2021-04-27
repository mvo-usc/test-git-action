# the different stages of this Dockerfile are meant to be built into separate images
# https://docs.docker.com/develop/develop-images/multistage-build/#stop-at-a-specific-build-stage
# https://docs.docker.com/compose/compose-file/#target


# https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact
ARG PHP_VERSION=8.0

# "php" stage
FROM php:${PHP_VERSION}-fpm-alpine AS api_platform_php

# persistent / runtime deps
RUN apk add --no-cache \
		acl \
		fcgi \
		file \
		gettext \
		git \
	;

ARG APCU_VERSION=5.1.19
RUN set -eux; \
	apk add --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		icu-dev \
		libzip-dev \
		postgresql-dev \
		zlib-dev \
	; \
	\
	docker-php-ext-configure zip; \
	docker-php-ext-install -j$(nproc) \
		intl \
		pdo_pgsql \
		zip \
	; \
	pecl install \
		apcu-${APCU_VERSION} \
	; \
	pecl clear-cache; \
	docker-php-ext-enable \
		apcu \
		opcache \
	; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --no-cache --virtual .api-phpexts-rundeps $runDeps; \
	\
	apk del .build-deps

RUN apk add --no-cache --virtual .user-deps \
            shadow \
      && usermod -u 1001 www-data \
	  && groupmod -g 121 www-data \
	  && apk del --no-cache .user-deps


###> recipes ###
###< recipes ###

COPY --from=composer:2.0 /usr/bin/composer /usr/bin/composer

RUN ln -s $PHP_INI_DIR/php.ini-production $PHP_INI_DIR/php.ini
COPY .docker/phpfpm/conf.d/api-platform.prod.ini $PHP_INI_DIR/conf.d/api-platform.ini

COPY .docker/phpfpm/php-fpm.d/zz-docker.conf /usr/local/etc/php-fpm.d/zz-docker.conf

VOLUME /var/run/php


WORKDIR /srv/api

# build for production
ARG APP_ENV=prod

# copy only specifically what we need
RUN ls -la
RUN pwd
COPY --chown="www-data:www-data" public public/

RUN set -eux; \
	rm -rf var; \
	mkdir -p var/cache var/log; sync

COPY .docker/phpfpm/entirypoint/docker-entrypoint.sh /usr/local/bin/docker-entrypoint
RUN chmod +x /usr/local/bin/docker-entrypoint

ENV SYMFONY_PHPUNIT_VERSION=9

ENTRYPOINT ["docker-entrypoint"]

ENV PHP_FPM_PM_LISTEN=127.0.0.1:9000
ENV PHP_FPM_PM=dynamic
ENV PHP_FPM_PM_MAX_CHILDREN=5
ENV PHP_FPM_PM_START_SERVERS=2
ENV PHP_FPM_PM_MIN_SPARE_SERVERS=1
ENV PHP_FPM_PM_MAX_SPARE_SERVERS=3
ENV PHP_FPM_PM_PROCESS_IDLE_TIMEOUT=10
ENV PHP_FPM_PM_MAX_REQUESTS=0
ENV PHP_FPM_PM_LOG_LEVEL=notice
ENV PHP_FPM_ACCESS_FORMAT %R - %u %t \\\"%m %r\\\" %s
# set to /dev/stderr to enable access request logging
ENV PHP_FPM_ACCESS_LOG=/dev/null

VOLUME /srv/api/var
VOLUME /srv/api

RUN chown -R www-data:www-data /srv/api
RUN chown -R www-data:www-data /usr/local/etc
RUN mkdir -p /run/php && chown -R www-data:www-data /run

USER www-data:www-data

WORKDIR /srv/api
EXPOSE 9000

CMD ["php-fpm"]
