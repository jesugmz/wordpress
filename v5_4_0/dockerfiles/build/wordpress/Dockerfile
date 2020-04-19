ARG WP_VERSION=5.4.0-fpm

FROM wordpress:$WP_VERSION AS base

RUN apt-get update -qq && apt-get install -qq -y --no-install-recommends --no-install-suggests \
        cron \
        supervisor \
    && mv /usr/src/wordpress/* /var/www/html \
    # clean up
    && rm -rf /tmp/* /var/lib/apt/lists/* /etc/supervisor/conf.d/ /usr/src/* /usr/local/bin/docker-entrypoint.sh \
    # smoke tests
    && supervisord --version

COPY ./config/php.ini /usr/local/etc/php/conf.d/10-base.ini

WORKDIR /var/www/html


#
# dev stage
#
FROM base AS dev

RUN pecl install \
        xdebug \
    && docker-php-ext-enable \
        xdebug \
    # smoke tests
    && set -ex \
    && ( \
      o=$(php -m | grep "xdebug") \
      && test "$o" = "xdebug" \
    )


#
# prod stage
#
FROM base AS prod

RUN groupadd --gid 2000 wordpress \
    && adduser --shell /bin/true --uid 2000 --gid 2000 --disabled-password --no-create-home --gecos '' wordpress \
    && chown -R wordpress:wordpress /var/www/ \
    # remove world-writeable permissions except for /tmp/
    && find / -xdev -type d -perm /0002 -exec chmod o-w {} + \
    && find / -xdev -type f -perm /0002 -exec chmod o-w {} + \
    && chmod 777 /tmp/ \
    && chown wordpress:root /tmp/ \
    # remove unnecessary accounts, excluding current wordpress and root users
    && sed -i -r '/^(wordpress|root|nobody)/!d' /etc/group \
    && sed -i -r '/^(wordpress|root|nobody)/!d' /etc/passwd \
    # remove interactive login shell for everybody
    && sed -i -r 's#^(.*):[^:]*$#\1:/sbin/nologin#' /etc/passwd \
    # disable password login for everybody
    && ( \
      while IFS=: read -r username _; do passwd -l "$username"; done < /etc/passwd || true \
    ) \
    # remove temporal shadow, passwd and group files
    && find /bin /etc /lib /sbin /usr -xdev -type f -regex ".*-$" -exec rm -f {} + \
    # ensure system dirs are owned by root and not writable by anybody else
    && find /bin /etc /lib /sbin /usr -xdev -type d -exec chown root:root {} \; -exec chmod 0755 {} \; \
    # remove suid & sgid files
    && find /bin /etc /lib /sbin /usr -xdev -type f -a \( -perm /4000 -o -perm /2000 \) -delete \
    # remove apt package manager
    && find / -type f -iname "*apt*" -xdev -delete \
    && find / -type d -iname "*apt*" -print0 -xdev | xargs -0 rm -rf -- \
    # remove any symlinks that we broke during previous steps
    && find /bin /etc /lib /sbin /usr -xdev -type l -exec test ! -e {} \; -delete \
    && rm -rf /root /etc/fstab

COPY ./config/php-fpm.conf /usr/local/etc/php-fpm.conf
COPY ./config/www.conf /usr/local/etc/php-fpm.d/www.conf
COPY ./config/php.ini /usr/local/etc/php.ini
COPY ./config/supervisord.conf /etc/supervisor/supervisord.conf
COPY --chown=wordpress:wordpress ./config/wp-config.php /var/www/html/wp-config.php
