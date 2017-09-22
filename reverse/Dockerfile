FROM alpine:3.6

ENV UID=991 GID=991

ARG NGINX_VERSION=1.13.5
ARG OPENSSL_VERSION=1.1.0f
ARG PHP_VER=7.1.9
ARG LIBICONV_VERSION=1.15
ARG GPG_NGINX="B0F4 2533 73F8 F6F5 10D4  2178 520A 9993 A1C0 52F8"
ARG BUILD_CORES

ARG PHP_MIRROR=http://ch1.php.net

ARG NGINX_MODULES=" \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_gzip_static_module \
    --with-http_stub_status_module \
    --with-file-aio \
    --with-threads \
    --with-pcre-jit \
    --without-http_ssi_module \
    --without-http_scgi_module \
    --without-http_uwsgi_module \
    --without-http_geo_module \
    --without-http_split_clients_module \
    --without-http_memcached_module \
    --without-http_empty_gif_module \
    --without-http_browser_module"

ARG NGINX_3RD_PARTY_MODULES=" \
    --add-module=/tmp/headers-more-nginx-module \
    --add-module=/tmp/ngx_brotli \
    --add-module=/tmp/nginx-ct"

    ARG PHP_CONF=" \
        --prefix=/usr \
        --libdir=/usr/lib/php \
        --datadir=/usr/share/php \
        --sysconfdir=/php/etc \
        --localstatedir=/php/var \
        --with-pear=/usr/share/php \
        --with-config-file-scan-dir=/php/conf.d \
        --with-config-file-path=/php \
        --with-pic \
        --disable-short-tags \
        --without-readline \
        --enable-bcmath=shared \
        --enable-fpm \
        --disable-cgi \
        --enable-mysqlnd \
        --enable-mbstring \
        --with-curl \
        --with-libedit \
        --with-openssl \
        --with-iconv=/usr/local \
        --with-gd \
        --with-jpeg-dir \
        --with-png-dir \
        --with-webp-dir \
        --with-xpm-dir=no \
        --with-freetype-dir \
        --enable-gd-native-ttf \
        --disable-gd-jis-conv \
        --with-zlib"

    ARG PHP_EXT_LIST=" \
        mysqli \
        ctype \
        dom \
        json \
        xml \
        libxml2 \
        mbstring \
        posix \
        xmlwriter \
        zip \
        zlib \
        sqlite3 \
        pdo_sqlite \
        pdo_pgsql \
        pdo_mysql \
        pcntl \
        curl \
        fileinfo \
        bz2 \
        intl \
        mcrypt \
        openssl \
        ldap \
        simplexml \
        pgsql \
        ftp \
        exif \
        gmp"

RUN NB_CORES=${BUILD_CORES-$(getconf _NPROCESSORS_CONF)} \

# Update system
 && apk -U upgrade \

# Installing runtime dependencies
 && apk add \
    ${BUILD_DEPS} \
    pcre \
    curl \
    zlib \
    libgcc \
    libstdc++ \
    jemalloc \
    su-exec \
    s6 \
    jq \
    libressl \
    bind-tools \
    libxml2 \
    libwebp \
    freetype \
    openldap \
    gmp \
    libmcrypt \
    bzip2-dev \
    icu \
    libpq \

# Installing build dependencies
 && apk add -t build-dependencies \
    build-base \
    linux-headers \
    ca-certificates \
    automake \
    autoconf \
    git \
    go \
    g++ \
    libc-dev \
    jemalloc-dev \
    tar \
    curl-dev \
    libtool \
    pcre-dev \
    zlib-dev \
    binutils \
    gnupg \
    cmake \
    libxml2-dev \
    pcre-dev \
    zlib-dev \
    pkgconf \
    patch \
    libwebp-dev \
    libjpeg-turbo-dev \
    libpng-dev \
    libxpm-dev \
    libedit-dev \
    libbz2 \
    sqlite-dev \
    gd-dev \
    freetype-dev \
    openldap-dev \
    gmp-dev \
    libmcrypt-dev \
    icu-dev \
    postgresql-dev \

# Brotli
 && cd /tmp && git clone https://github.com/bagder/libbrotli --depth=1 \
 && cd libbrotli && ./autogen.sh && ./configure && make -j ${NB_CORES} && make install \
 && cd /tmp && git clone https://github.com/google/ngx_brotli --depth=1 \
 && cd ngx_brotli && git submodule update --init && cd /tmp \

# Headers More
 && git clone https://github.com/openresty/headers-more-nginx-module --depth=1 \

# nginx-ct
&& git clone https://github.com/grahamedgecombe/nginx-ct --depth=1 \

# OpenSSL
 && OPENSSL_TARBALL="openssl-${OPENSSL_VERSION}.tar.gz" \
 && wget -q https://www.openssl.org/source/${OPENSSL_TARBALL} \
 && tar xzf ${OPENSSL_TARBALL} \
 && cd openssl-${OPENSSL_VERSION} \
 && wget https://raw.githubusercontent.com/cloudflare/sslconfig/master/patches/openssl__1.1.0_chacha20_poly1305.patch \
 && patch -p1 < openssl__1.1.0_chacha20_poly1305.patch \
 && cd /tmp \

# Nginx tarball checking
 && NGINX_TARBALL="nginx-${NGINX_VERSION}.tar.gz" \
 && wget -q https://nginx.org/download/${NGINX_TARBALL} \
 && echo "Verifying ${NGINX_TARBALL} using GPG..." \
 && wget -q https://nginx.org/download/${NGINX_TARBALL}.asc \
 && wget -q https://nginx.org/keys/mdounin.key \
 && gpg --import mdounin.key \
 && FINGERPRINT="$(LANG=C gpg --verify ${NGINX_TARBALL}.asc ${NGINX_TARBALL} 2>&1 \
  | sed -n "s#Primary key fingerprint: \(.*\)#\1#p")" \
 && if [ -z "${FINGERPRINT}" ]; then echo "Warning! Invalid GPG signature!" && exit 1; fi \
 && if [ "${FINGERPRINT}" != "${GPG_NGINX}" ]; then echo "Warning! Wrong GPG fingerprint!" && exit 1; fi \
 && echo "All seems good, now unpacking ${NGINX_TARBALL}..." \
 && tar xzf ${NGINX_TARBALL} && cd nginx-${NGINX_VERSION} \

# Nginx patch : dynamic TLS records
 && wget -q https://raw.githubusercontent.com/cujanovic/nginx-dynamic-tls-records-patch/master/nginx__dynamic_tls_records_1.13.0%2B.patch -O dynamic_records.patch \
 && patch -p1 < dynamic_records.patch \

# Nginx compilation
 && ./configure \
    --prefix=/nginx \
    --sbin-path=/usr/local/sbin/nginx \
    --with-cc-opt="-O3 -fPIE -fstack-protector-strong -D_FORTIFY_SOURCE=2 -Wformat -Werror=format-security -Wno-deprecated-declarations" \
    --with-ld-opt="-lrt -ljemalloc -Wl,-Bsymbolic-functions -Wl,-z,relro" \
    --with-openssl-opt='no-async enable-ec_nistp_64_gcc_128 no-shared no-ssl2 no-ssl3 no-comp no-idea no-weak-ssl-ciphers -DOPENSSL_NO_HEARTBEATS -O3 -fPIE -fstack-protector-strong -D_FORTIFY_SOURCE=2' \
    --with-openssl=/tmp/openssl-${OPENSSL_VERSION} \
    --http-log-path=/nginx/log/nginx_access.log \
    --error-log-path=/nginx/log/nginx_error.log \
    --pid-path=/nginx/run/nginx.pid \
    --lock-path=/nginx/run/nginx.lock \
    --user=reverse --group=reverse \
    ${NGINX_MODULES} \
    ${NGINX_3RD_PARTY_MODULES} \
 && make -j ${NB_CORES} && make install && make clean \
 && strip -s /usr/local/sbin/nginx \

 ### GNU Libiconv installation
 && wget https://ftp.gnu.org/pub/gnu/libiconv/libiconv-${LIBICONV_VERSION}.tar.gz -O /tmp/libiconv-${LIBICONV_VERSION}.tar.gz \
 && tar xzf /tmp/libiconv-${LIBICONV_VERSION}.tar.gz -C /tmp \
 && cd /tmp/libiconv-${LIBICONV_VERSION} \
  && ./configure --prefix=/usr/local \
  && make -j$NB_CORES && make install && libtool --finish /usr/local/lib \


 ### PHP installation
  && wget ${PHP_MIRROR}/get/php-${PHP_VER}.tar.gz/from/this/mirror -O /tmp/php-${PHP_VER}.tar.gz \
  && wget ${PHP_MIRROR}/get/php-${PHP_VER}.tar.gz.asc/from/this/mirror -O /tmp/php-${PHP_VER}.tar.gz.asc \
  && mkdir -p /php/conf.d \
  && tar xzvf /tmp/php-${PHP_VER}.tar.gz -C /tmp \
  && mv /tmp/php-${PHP_VER} /tmp/php \
  && cd /tmp/php \
  && ./configure CFLAGS="-O3 -fstack-protector-strong" ${PHP_CONF} \
  && make -j ${NB_CORES} \
  && make install \

# Lego
 && mkdir -p /tmp/go/bin \
 && export GOPATH=/tmp/go \
 && export GOBIN=$GOPATH/bin \
 && git config --global http.https://gopkg.in.followRedirects true \
 && go get github.com/xenolf/lego \
 && mv /tmp/go/bin/lego /usr/local/bin/lego \

# ct-submit
 && go get github.com/grahamedgecombe/ct-submit \
 && mv /tmp/go/bin/ct-submit /usr/local/bin/ct-submit \

# Clean
 && apk del build-dependencies \
 && rm -rf /tmp/* /var/cache/apk/* /root/.gnupg

COPY rootfs /

RUN chmod +x /usr/local/bin/startup /etc/s6.d/*/*

EXPOSE 8000 8443

ENTRYPOINT ["/usr/local/bin/startup"]
CMD ["/bin/s6-svscan", "/etc/s6.d"]
