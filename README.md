# laravel-sail-docker-xdebug

`cp -r vendor/laravel/sail/runtimes/8.0 ./resources/docker/`

```
version: '3'
services:
    laravel.test:
        build:
            context: ./resources/docker/
            dockerfile: Dockerfile
            args:
                WWWGROUP: '${WWWGROUP}'
                XDEBUG: ${APP_DEBUG}

...
```

```
FROM ubuntu:20.04

LABEL maintainer="Taylor Otwell"

ARG WWWGROUP
ARG XDEBUG

WORKDIR /var/www/html

ENV DEBIAN_FRONTEND noninteractive
ENV TZ=UTC

RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

RUN apt-get update \
    && apt-get install -y gnupg gosu curl ca-certificates zip unzip git supervisor sqlite3 libcap2-bin \
    && mkdir -p ~/.gnupg \
    && echo "disable-ipv6" >> ~/.gnupg/dirmngr.conf \
    && apt-key adv --homedir ~/.gnupg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys E5267A6C \
    && apt-key adv --homedir ~/.gnupg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C300EE8C \
    && echo "deb http://ppa.launchpad.net/ondrej/php/ubuntu focal main" > /etc/apt/sources.list.d/ppa_ondrej_php.list \
    && apt-get update \
    && apt-get install -y php8.0-cli php8.0-dev \
       php8.0-pgsql php8.0-sqlite3 php8.0-gd \
       php8.0-curl php8.0-memcached \
       php8.0-imap php8.0-mysql php8.0-mbstring \
       php8.0-xml php8.0-zip php8.0-bcmath php8.0-soap \
       php8.0-intl php8.0-readline \
       php8.0-msgpack php8.0-igbinary php8.0-ldap \
       php8.0-redis

RUN if [ ${XDEBUG} ] ; then \
    apt-get install -y php-xdebug \
    && echo "[XDebug]" > /etc/php/8.0/cli/php.ini \
    && echo "zend_extension="$(find /usr/lib/php/20200930/ -name xdebug.so)" > /etc/php/8.0/cli/php.ini" \
    && echo "xdebug.mode = debug" >> /etc/php/8.0/cli/php.ini \
    && echo "xdebug.start_with_request = yes" >> /etc/php/8.0/cli/php.ini \
    && echo "xdebug.discover_client_host = true" >> /etc/php/8.0/cli/php.ini ;\
fi;

RUN php -r "readfile('http://getcomposer.org/installer');" | php -- --install-dir=/usr/bin/ --filename=composer \
    && curl -sL https://deb.nodesource.com/setup_15.x | bash - \
    && apt-get install -y nodejs \
    && apt-get -y autoremove \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*


RUN setcap "cap_net_bind_service=+ep" /usr/bin/php8.0

RUN groupadd --force -g $WWWGROUP sail
RUN useradd -ms /bin/bash --no-user-group -g $WWWGROUP -u 1337 sail

COPY start-container /usr/local/bin/start-container
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY php.ini /etc/php/8.0/cli/conf.d/99-sail.ini
RUN chmod +x /usr/local/bin/start-container

EXPOSE 8000

ENTRYPOINT ["start-container"]
```

```
$ ./vendor/bin/sail stop
$ ./vendor/bin/sail up --build -d
```

```
$ ./vendor/bin/sail php -v
PHP 8.0.0 (cli) (built: Nov 27 2020 12:26:22) ( NTS )
Copyright (c) The PHP Group
Zend Engine v4.0.0-dev, Copyright (c) Zend Technologies
    with Zend OPcache v8.0.0, Copyright (c), by Zend Technologies
    with Xdebug v3.0.1, Copyright (c) 2002-2020, by Derick Rethans
``` 
