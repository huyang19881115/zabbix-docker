# syntax=docker/dockerfile:1
ARG MAJOR_VERSION=6.4
ARG ZBX_VERSION=${MAJOR_VERSION}.12
ARG BUILD_BASE_IMAGE=zabbix-build-pgsql:ubuntu-${ZBX_VERSION}

FROM ${BUILD_BASE_IMAGE} AS builder

FROM ubuntu:jammy

ARG MAJOR_VERSION
ARG ZBX_VERSION
ARG ZBX_SOURCES=https://git.zabbix.com/scm/zbx/zabbix.git

ENV TERM=xterm \
    ZBX_VERSION=${ZBX_VERSION} ZBX_SOURCES=${ZBX_SOURCES} \
    MIBDIRS=/var/lib/mibs/ietf:/var/lib/mibs/iana:/usr/share/snmp/mibs:/var/lib/zabbix/mibs MIBS=+ALL \
    NMAP_PRIVILEGED=""

LABEL org.opencontainers.image.authors="Alexey Pustovalov <alexey.pustovalov@zabbix.com>" \
      org.opencontainers.image.description="Zabbix server with PostgreSQL database support" \
      org.opencontainers.image.documentation="https://www.zabbix.com/documentation/${MAJOR_VERSION}/manual/installation/containers" \
      org.opencontainers.image.licenses="GPL v2.0" \
      org.opencontainers.image.source="${ZBX_SOURCES}" \
      org.opencontainers.image.title="Zabbix server (PostgreSQL)" \
      org.opencontainers.image.url="https://zabbix.com/" \
      org.opencontainers.image.vendor="Zabbix LLC" \
      org.opencontainers.image.version="${ZBX_VERSION}"

STOPSIGNAL SIGTERM

COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/zabbix_server/zabbix_server", "/usr/sbin/zabbix_server"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/zabbix_get/zabbix_get", "/usr/bin/zabbix_get"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/zabbix_sender/zabbix_sender", "/usr/bin/zabbix_sender"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/conf/zabbix_server.conf", "/etc/zabbix/zabbix_server.conf"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/database/postgresql/create_server.sql.gz", "/usr/share/doc/zabbix-server-postgresql/create.sql.gz"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/database/postgresql/timescaledb/schema.sql", "/usr/share/doc/zabbix-server-postgresql/timescaledb.sql"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/database/postgresql/timescaledb/option-patches/with-compression/*.sql", "/usr/share/doc/zabbix-server-postgresql/option-patches/with-compression/"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/database/postgresql/timescaledb/option-patches/without-compression/*.sql", "/usr/share/doc/zabbix-server-postgresql/option-patches/without-compression/"]

RUN --mount=type=cache,target=/var/lib/apt/,sharing=locked \
    set -eux && \
    echo "#!/bin/sh\nexit 101" > /usr/sbin/policy-rc.d && \
    INSTALL_PKGS="bash \
            tini \
            traceroute \
            nmap \
            tzdata \
            ca-certificates \
            iputils-ping \
            traceroute \
            fping \
            libcurl4 \
            libevent-2.1 \
            libevent-pthreads-2.1 \
            libopenipmi0 \
            libpcre2-8-0 \
            libpq5 \
            libsnmp40 \
            libssh-4 \
            libssl3 \
            libxml2 \
            postgresql-client \
            snmp-mibs-downloader \
            gzip \
            unixodbc" && \
    apt-get -y update && \
    DEBIAN_FRONTEND=noninteractive apt-get -y \
            --no-install-recommends install \
        ${INSTALL_PKGS} && \
    groupadd \
            --system \
            --gid 1995 \
        zabbix && \
    useradd \
            --system \
            --comment "Zabbix monitoring system" \
            -g zabbix \
            -G dialout \
            --uid 1997 \
            --shell /sbin/nologin \
            --home-dir /var/lib/zabbix/ \
        zabbix && \
    chgrp zabbix /usr/bin/nmap && \
    setcap cap_net_raw+eip /usr/bin/nmap && \
    mkdir -p /etc/zabbix && \
    mkdir -p /var/lib/zabbix && \
    mkdir -p /usr/lib/zabbix/alertscripts && \
    mkdir -p /usr/lib/zabbix/dbscripts && \
    mkdir -p /var/lib/zabbix/dbscripts && \
    mkdir -p /var/lib/zabbix/enc && \
    mkdir -p /var/lib/zabbix/export && \
    mkdir -p /usr/lib/zabbix/externalscripts && \
    mkdir -p /var/lib/zabbix/mibs && \
    mkdir -p /var/lib/zabbix/modules && \
    mkdir -p /var/lib/zabbix/snmptraps && \
    mkdir -p /var/lib/zabbix/ssh_keys && \
    mkdir -p /var/lib/zabbix/ssl && \
    mkdir -p /var/lib/zabbix/ssl/certs && \
    mkdir -p /var/lib/zabbix/ssl/keys && \
    mkdir -p /var/lib/zabbix/ssl/ssl_ca && \
    mkdir -p /usr/share/doc/zabbix-server-postgresql && \
    chown --quiet -R zabbix:root /etc/zabbix/ /var/lib/zabbix/ && \
    chgrp -R 0 /etc/zabbix/ /var/lib/zabbix/ && \
    chmod -R g=u /etc/zabbix/ /var/lib/zabbix/ && \
    apt-get -y autoremove && \
    apt-get -y clean

EXPOSE 10051/TCP

WORKDIR /var/lib/zabbix

VOLUME ["/var/lib/zabbix/snmptraps", "/var/lib/zabbix/export"]

COPY ["docker-entrypoint.sh", "/usr/bin/"]

ENTRYPOINT ["/usr/bin/tini", "--", "/usr/bin/docker-entrypoint.sh"]

USER 1997

CMD ["/usr/sbin/zabbix_server", "--foreground", "-c", "/etc/zabbix/zabbix_server.conf"]
