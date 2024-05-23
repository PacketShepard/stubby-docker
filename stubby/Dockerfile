FROM debian:bookworm as openssl

ENV VERSION_OPENSSL=openssl-3.0.13 \
    SHA256_OPENSSL=88525753f79d3bec27d2fa7c66aa0b92b3aa9498dafd93d7cfa4b3780cdae313 \
    SOURCE_OPENSSL=https://www.openssl.org/source/ \
    OPGP_OPENSSL=EFC0A467D613CB83C7ED6D30D894E2CE8B3D79F5

WORKDIR /tmp/src
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN set -e -x && \
    build_deps="build-essential ca-certificates curl dirmngr gnupg libidn2-dev libssl-dev" && \
    DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y --no-install-recommends \
        $build_deps && \
    curl -L $SOURCE_OPENSSL$VERSION_OPENSSL.tar.gz -o openssl.tar.gz && \
    echo "${SHA256_OPENSSL} ./openssl.tar.gz" | sha256sum -c - && \
    curl -L $SOURCE_OPENSSL$VERSION_OPENSSL.tar.gz.asc -o openssl.tar.gz.asc && \
    GNUPGHOME="$(mktemp -d)" && \
    export GNUPGHOME && \
    gpg --no-tty --keyserver keys.openpgp.org --recv-keys "$OPGP_OPENSSL" && \
    gpg --batch --verify openssl.tar.gz.asc openssl.tar.gz && \
    tar xzf openssl.tar.gz && \
    cd "${VERSION_OPENSSL}" && \
    /bin/sh -c 'if gcc -dM -E - </dev/null | grep -q __SIZEOF_INT128__; then export ECFLAG="enable-ec_nistp_64_gcc_128"; else export ECFLAG=""; fi' && \
    ./config \
        -Wl,-rpath=/opt/openssl/lib \
        --prefix=/opt/openssl \
        --openssldir=/opt/openssl \
        $ECFLAG \
        -DOPENSSL_NO_HEARTBEATS \
        no-weak-ssl-ciphers \
        no-ssl2 \
        no-ssl3 \
        shared \
        -fstack-protector-strong && \
    make depend && \
    make && \
    make install_sw

FROM debian:bookworm as stubby
LABEL maintainer="Matthew Vance"

ENV VERSION_GETDNS=getdns-1.7.3 \
    SHA256_GETDNS=f1404ca250f02e37a118aa00cf0ec2cbe11896e060c6d369c6761baea7d55a2c \
    SOURCE_GETDNS=https://getdnsapi.net/dist/ \
	OPGP_GETDNS=DC34EE5DB2417BCC151E5100E5F8F8212F77A498

WORKDIR /tmp/src
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

COPY --from=openssl /opt/openssl /opt/openssl

RUN set -e -x && \
    build_deps="autoconf build-essential curl dirmngr gnupg check cmake dh-autoreconf libssl-dev libyaml-dev make m4" && \
    debian_frontend=noninteractive apt-get update && apt-get install -y --no-install-recommends \
      ${build_deps} \
      ca-certificates \
      dns-root-data \
      libyaml-0-2 && \
    debian_frontend=noninteractive apt-get update && apt-get install -y --no-install-recommends check cmake && \
    #git clone https://github.com/getdnsapi/getdns.git && \
    #cd getdns && \
    #git checkout "${VERSION_GETDNS}" && \
    #git submodule update --init && \
	curl -L $SOURCE_GETDNS$VERSION_GETDNS.tar.gz -o getdns.tar.gz && \
    echo "${SHA256_GETDNS} ./getdns.tar.gz" | sha256sum -c - && \
    curl -L $SOURCE_GETDNS$VERSION_GETDNS.tar.gz.asc -o getdns.tar.gz.asc && \
    GNUPGHOME="$(mktemp -d)" && \
    export GNUPGHOME && \
    gpg --no-tty --keyserver keys.openpgp.org --recv-keys "$OPGP_GETDNS" && \
    gpg --batch --verify getdns.tar.gz.asc getdns.tar.gz && \
    tar xzf getdns.tar.gz && \
    cd "${VERSION_GETDNS}" && \
    mkdir build && \
    cd build && \
    cmake \
        -DBUILD_STUBBY=ON \
        -DENABLE_STUB_ONLY=ON \
        -DCMAKE_INSTALL_PREFIX=/opt/stubby \
        -DOPENSSL_INCLUDE_DIR=/opt/openssl \
        -DOPENSSL_CRYPTO_LIBRARY=/opt/openssl/lib64/libcrypto.so \
        -DOPENSSL_SSL_LIBRARY=/opt/openssl/lib64/libssl.so \
        -DUSE_LIBIDN2=OFF \
        -DBUILD_LIBEV=OFF \
        -DBUILD_LIBEVENT2=OFF \
        -DBUILD_LIBUV=OFF ..&& \
    cmake .. && \
    make && \
    make install

FROM debian:bookworm

COPY --from=openssl /opt/openssl /opt/openssl
COPY --from=stubby /opt/stubby /opt/stubby
COPY stubby.yml-example /opt/stubby/etc/stubby/stubby.yml

ENV PATH /opt/stubby/bin:$PATH

RUN set -e -x && \
    debian_frontend=noninteractive apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      dns-root-data \
      ldnsutils \
      libyaml-0-2 && \
    groupadd -r stubby && \
    useradd --no-log-init -r -g stubby stubby && \
    rm -rf \
      /tmp/* \
      /var/tmp/* \
      /var/lib/apt/lists/*

WORKDIR /opt/stubby

EXPOSE 8053/udp

USER stubby:stubby

HEALTHCHECK --interval=30s --timeout=30s --start-period=10s CMD drill @127.0.0.1 -p 8053 cloudflare.com || exit 1

CMD ["/opt/stubby/bin/stubby", "-C", "/opt/stubby/etc/stubby/stubby.yml"]
