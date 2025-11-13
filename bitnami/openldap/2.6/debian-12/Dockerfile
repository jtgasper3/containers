# Copyright Broadcom, Inc. All Rights Reserved.
# SPDX-License-Identifier: APACHE-2.0

# Stage 1: Build OpenLDAP from source
FROM debian:bookworm AS builder

ARG OPENLDAP_VERSION=2.6.10

RUN apt update
RUN apt install -y \
    autoconf \
    automake \
    bison \
    build-essential \
    ca-certificates \
    flex \
    groff \
    libtool \
    pkg-config \
    wget
RUN apt install -y \
    libargon2-dev \
    libltdl-dev \
    libdb-dev \
    libsasl2-dev \
    libgnutls28-dev libssl-dev \
    libiodbc2-dev \
    libkrb5-dev \
    libperl-dev

#  Try the above with --no-install-recommends
WORKDIR /build/openldap

RUN wget https://www.openldap.org/software/download/OpenLDAP/openldap-release/openldap-${OPENLDAP_VERSION}.tgz && \
    tar -xzf openldap-${OPENLDAP_VERSION}.tgz --strip-components=1 && \
    rm openldap-${OPENLDAP_VERSION}.tgz

# Configure with Bitnami-style layout
RUN CPPFLAGS="-I/usr/include/iodbc" ./configure \
    --prefix=/opt/bitnami/openldap \
    --exec-prefix=/opt/bitnami/openldap \
    --libexecdir=/opt/bitnami/openldap/sbin \
    --with-subdir=../lib/openldap \
    --enable-modules \
    --enable-crypt \
    --enable-spasswd \
    --enable-slapi \
    --enable-asyncmeta \
    --enable-dnssrv \
    --enable-ldap \
    --enable-meta \
    --enable-null \
    --enable-passwd \
    --enable-perl \
    --enable-relay \
    --enable-sock \
    --enable-sql \
    --enable-overlays=mod

RUN make depend
RUN make -j$(nproc) && make install

# Build and install some additional modules
RUN PROJECT_ROOT=$(pwd) && \
  for module in autogroup lastbind passwd/pbkdf2 passwd/sha2; do \
    echo "Building and installing module: $module" && \
    cd "$PROJECT_ROOT/contrib/slapd-modules/$module" &&  \
    make && make install; \
  done

# Remove build related files
RUN find /opt/bitnami/openldap -name "*.la" -type f -delete && \
    find /opt/bitnami/openldap -name "*.a" -type f -delete

# Shuffle some files around to match Bitnami layout
RUN rm -rf /opt/bitnami/openldap/share/man/ && \
    mv /opt/bitnami/openldap/lib/openldap/ldap.conf* /opt/bitnami/openldap/etc/ && \
    mv /opt/bitnami/openldap/lib/openldap/slapd.ldif /opt/bitnami/openldap/share/ && \
    mv /opt/bitnami/openldap/lib/openldap/schema/ /opt/bitnami/openldap/etc/ && \
    mkdir /opt/bitnami/openldap/lib/openldap/certs && \
    cp -r /usr/local/libexec/ /opt/bitnami/openldap/libexec/


  # Stage 2: Bitnami image
FROM docker.io/bitnami/minideb:bookworm

ARG TARGETARCH

LABEL org.opencontainers.image.base.name="docker.io/bitnami/minideb:bookworm" \
      org.opencontainers.image.created="2025-09-07T02:04:16Z" \
      org.opencontainers.image.description="Application packaged by John Gasper." \
      org.opencontainers.image.documentation="https://github.com/bitnami/containers/tree/main/bitnami/openldap/README.md" \
      org.opencontainers.image.source="https://github.com/bitnami/containers/tree/main/bitnami/openldap" \
      org.opencontainers.image.title="openldap" \
      org.opencontainers.image.vendor="John Gasper" \
      org.opencontainers.image.version="2.6.10"

ENV HOME="/" \
    OS_ARCH="${TARGETARCH:-amd64}" \
    OS_FLAVOUR="debian-12" \
    OS_NAME="linux"

COPY prebuildfs /
SHELL ["/bin/bash", "-o", "errexit", "-o", "nounset", "-o", "pipefail", "-c"]
# Install required system packages and dependencies
RUN install_packages \
    libargon2-1 libcap2-bin libcom-err2 libcrypt1 libgssapi-krb5-2 \
    libk5crypto3 libkeyutils1 libkrb5-3 libkrb5support0 libltdl7 \
    libnsl2 libnss3-tools libodbc2 libperl5.36 libsasl2-2 libssl3 \
    libtirpc3 libwrap0 mdbtools procps psmisc

# Copy built OpenLDAP from builder
COPY --from=builder /opt/bitnami/openldap /opt/bitnami/openldap

RUN apt-get update && apt-get upgrade -y && \
    apt-get clean && rm -rf /var/lib/apt/lists /var/cache/apt/archives
RUN chmod g+rwX /opt/bitnami
RUN find / -perm /6000 -type f -exec chmod a-s {} \; || true

COPY rootfs/ /
RUN /opt/bitnami/scripts/openldap/postunpack.sh

ENV APP_VERSION="2.6.10" \
    BITNAMI_APP_NAME="openldap" \
    IMAGE_REVISION="5" \
    PATH="/opt/bitnami/openldap/bin:/opt/bitnami/openldap/sbin:$PATH"

EXPOSE 1389 1636

USER 1001
ENTRYPOINT [ "/opt/bitnami/scripts/openldap/entrypoint.sh" ]
CMD [ "/opt/bitnami/scripts/openldap/run.sh" ]