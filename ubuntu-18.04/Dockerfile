FROM ubuntu:bionic-20200807

LABEL name="nmoeini/redis:6-bionic-20200807" mainteiner="Navid Moeini <nmoeiny@yahoo.com>" version="0.0"

ARG DEBIAN_FRONTEND
ARG GOSU_VERSION
ARG SVC_USER
ARG REDIS_VERSION

ENV GOSU_VERSION=${GOSU_VERSION:-1.12} \
    SVC_USER=${SVC_USER:-redis} \
    DEBIAN_FRONTEND=${DEBIAN_FRONTEND:-noninteractive} \
    REDIS_VERSION=${REDIS_VERSION:-6.0.5}

RUN \
    \
##  Create the redis system user \
##  ------------------------------------------------------------------------  ## \
    groupadd --system --force --gid 999 "${SVC_USER}" \
    && useradd --system --uid 999 --gid "${SVC_USER}" --shell /bin/bash --create-home "${SVC_USER}"

RUN set -eux \
	&& savedAptMark="$(apt-mark showmanual)" \
    && apt-get update -y \
	&& apt-get install -y --no-install-recommends \
        ca-certificates \
        dirmngr \
        gnupg \
        wget \
	&& rm -rf /var/lib/apt/lists/* \
	&& dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')" \
	&& wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch" \
	&& wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& gpgconf --kill all \
	&& rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& apt-mark auto '.*' > /dev/null \
	&& [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark > /dev/null \
	&& apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu --version \
	&& gosu nobody true

ENV REDIS_DOWNLOAD_URL http://download.redis.io/releases/redis-"${REDIS_VERSION}".tar.gz
ENV REDIS_DOWNLOAD_SHA 42cf86a114d2a451b898fcda96acd4d01062a7dbaaad2801d9164a36f898f596

RUN set -eux \
	\
	&& savedAptMark="$(apt-mark showmanual)" \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends \
		gcc \
		libc6-dev \
		libssl-dev \
		make \
		wget \
	\
	&& wget -O redis.tar.gz "$REDIS_DOWNLOAD_URL" \
	&& echo "$REDIS_DOWNLOAD_SHA *redis.tar.gz" | sha256sum -c - \
	&& mkdir -p /usr/src/redis \
	&& tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
	&& rm redis.tar.gz \
	\
# disable Redis protected mode [1] as it is unnecessary in context of Docker
# (ports are not automatically exposed when running inside Docker, but rather explicitly by specifying -p / -P)
# [1]: https://github.com/antirez/redis/commit/edd4d555df57dc84265fdfb4ef59a4678832f6da
	&& grep -E '^ *createBoolConfig[(]"protected-mode",.*, *1 *,.*[)],$' /usr/src/redis/src/config.c \
	&& sed -ri 's!^( *createBoolConfig[(]"protected-mode",.*, *)1( *,.*[)],)$!\10\2!' /usr/src/redis/src/config.c \
	&& grep -E '^ *createBoolConfig[(]"protected-mode",.*, *0 *,.*[)],$' /usr/src/redis/src/config.c \
# for future reference, we modify this directly in the source instead of just supplying a default configuration flag because apparently "if you specify any argument to redis-server, [it assumes] you are going to specify everything"
# see also https://github.com/docker-library/redis/issues/4#issuecomment-50780840
# (more exactly, this makes sure the default behavior of "save on SIGTERM" stays functional by default)
	\
	&& export BUILD_TLS=yes \
	&& make -C /usr/src/redis -j "$(nproc)" all \
	&& make -C /usr/src/redis install \
	&& serverMd5="$(md5sum /usr/local/bin/redis-server | cut -d' ' -f1)"; export serverMd5 \
	&& find /usr/local/bin/redis* -maxdepth 0 \
		-type f -not -name redis-server \
		-exec sh -eux -c ' \
			md5="$(md5sum "$1" | cut -d" " -f1)"; \
			test "$md5" = "$serverMd5"; \
		' -- '{}' ';' \
		-exec ln -svfT 'redis-server' '{}' ';' \
	\
	&& rm -r /usr/src/redis \
	\
	&& apt-mark auto '.*' > /dev/null \
	&& [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark > /dev/null \
	&& find /usr/local -type f -executable -exec ldd '{}' ';' \
		| awk '/=>/ { print $(NF-1) }' \
		| sort -u \
		| xargs -r dpkg-query --search \
		| cut -d: -f1 \
		| sort -u \
		| xargs -r apt-mark manual \
	\
	&& redis-cli --version \
	&& redis-server --version \
##  Clear cache \
##  -----------------------------------------------------------------------  ## \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir /data && chown "${SVC_USER}":"${SVC_USER}" /data
VOLUME /data
WORKDIR /data

COPY docker-entrypoint.sh /usr/local/bin/

RUN chmod +x /usr/local/bin/docker-*

ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 6379
CMD ["redis-server"]