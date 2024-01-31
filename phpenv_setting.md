# Multiple PHP versions using `phpenv` and `php-build`

## Install dependecies

### Debian/Ubuntu users

```sh
sudo apt install \
  autoconf \
  bison \
  build-essential \
  libbz2-dev \
  libc-client-dev \
  libcurl4-gnutls-dev \
  libedit-dev \
  libgmp-dev \
  libjpeg-dev \
  libkrb5-dev \
  libmagickcore-dev \
  libmagickwand-dev \
  libmcrypt-dev \
  libmemcached-dev \
  libpng-dev \
  libreadline-dev \
  libonig-dev \
  libpq-dev \
  libssl-dev \
  libtidy-dev \
  libwebp-dev \
  libxml2-dev \
  libxpm-dev \
  libxslt-dev \
  libyaml-dev \
  libzip-dev \
  linux-libc-dev \
  pkg-config \
  re2c
```

### macOS users

```sh
brew install \
  autoconf \
  automake \
  bison \
  bzip2 \
  curl \
  gettext \
  gmp \
  icu4c \
  jpeg \
  krb5 \
  imagemagick \
  libedit \
  libiconv \
  libmcrypt \
  libmemcached \
  libpng \
  libxml2 \
  libxslt \
  libzip \
  libtool \
  make \
  openssl \
  re2c \
  readline \
  tidy-html5 \
  zlib
```

## Installation

### Install `phpenv`

```bash
git clone git@github.com:phpenv/phpenv.git ~/.phpenv
```

Then follow the [instruction from phpenv repo](https://github.com/phpenv/phpenv#basic-github-checkout).

### Install `php-build`

```bash
git clone git@github.com:php-build/php-build.git ~/.phpenv/plugins/php-build
```

### Install `config-add` plugin

```bash
git clone git@github.com:sergeyklay/phpenv-config-add.git ~/.phpenv/plugins/phpenv-config-add
```

### Install `pear-setup` plugin

```bash
git clone git@github.com:sergeyklay/phpenv-pear-setup.git ~/.phpenv/plugins/phpenv-pear-setup
```

## Usage

```sh
export PHP_BUILD_ZTS_ENABLE=on
export PHP_BUILD_XDEBUG_ENABLE=off
export PHP_BUILD_EXTRA_MAKE_ARGUMENTS=-j"$(getconf _NPROCESSORS_ONLN)"
export PHP_BUILD_TMPDIR=~/src/php
mkdir -p $PHP_BUILD_TMPDIR

php-build --ini development 7.4.0 $(phpenv root)/versions/7.4.0-zts-debug/
```

Example of script to print install instructions for multiple PHP versions:
```sh
for v in 7.0.33 7.1.30 7.2.20 7.3.7 7.4.0; do
  for z in on off; do
    for d in --enable-debug --disable-debug; do
      [ "$z" = "on" ]             && zts=zts   || zts=nts
      [ "$d" = "--enable-debug" ] && dbg=debug || dbg=release
      echo \
        PHP_BUILD_ZTS_ENABLE=$z \
        PHP_BUILD_XDEBUG_ENABLE=off \
        PHP_BUILD_EXTRA_MAKE_ARGUMENTS=-j"$(getconf _NPROCESSORS_ONLN)" \
        PHP_BUILD_CONFIGURE_OPTS="\"--disable-fpm --disable-phpdbg $d\"" \
        php-build --ini development $v $(phpenv root)/versions/$v-$zts-$dbg/
    done
  done
done
```

This will print lines like this:
```
PHP_BUILD_ZTS_ENABLE=on \
    PHP_BUILD_XDEBUG_ENABLE=off \
    PHP_BUILD_EXTRA_MAKE_ARGUMENTS=-j20 \
    PHP_BUILD_CONFIGURE_OPTS="--disable-fpm --disable-phpdbg --enable-debug" \
    php-build --ini development 7.4.0 $(phpenv root)/versions/7.4.0-zts-debug/
```

Feel free to disable any extension using `PHP_BUILD_CONFIGURE_OPTS` variable. For examle to disabe `gd`:
```diff
- PHP_BUILD_CONFIGURE_OPTS="--disable-fpm --disable-phpdbg --enable-debug"
+ PHP_BUILD_CONFIGURE_OPTS="--disable-fpm --disable-phpdbg --disable-gd --enable-debug"
```

### Notes for macOS

macOS users will need to create a symlink for openssl and customize `PHP_BUILD_CONFIGURE_OPTS`. For example:

Creating symlinks:
```sh
cd /usr/local/include
ln -s ../opt/openssl/include/openssl .
```

Customizing `PHP_BUILD_CONFIGURE_OPTS`:

```sh
export PHP_BUILD_CONFIGURE_OPTS="\
  --disable-fpm \
  --disable-phpdbg \
  --enable-debug \
  --with-bz2=$(brew --prefix bzip2) \
  --with-curl=$(brew --prefix curl) \
  --with-gettext=$(brew --prefix gettext) \
  --with-gmp=$(brew --prefix gmp) \
  --with-iconv=$(brew --prefix libiconv) \
  --with-icu-dir=$(brew --prefix icu4c) \
  --with-jpeg-dir=$(brew --prefix jpeg) \
  --with-libedit=$(brew --prefix libedit) \
  --with-libxml-dir=$(brew --prefix libxml2) \
  --with-libzip=$(brew --prefix libzip)
  --with-mcrypt=$(brew --prefix libmcrypt) \
  --with-png-dir=$(brew --prefix libpng) \
  --with-readline=$(brew --prefix readline) \
  --with-tidy=$(brew --prefix tidy-html5) \
  --with-xsl=$(brew --prefix libxslt) \
  --with-zlib=$(brew --prefix zlib)" \
  --with-kerberos
```

For PHP 7.0.* you'll need `buffio.h`: 

```sh
ln -s  $(brew --prefix tidy-html5)/include/tidybuffio.h $(brew --prefix tidy-html5)/include/buffio.h
```

For PHP >= 7.4 you'll need to specify library path for the folowing packages:
- openssl
- icu4c
- libxml2

```sh
PKG_CONFIG_PATH=""
for pkg in openssl icu4c libxml2; do
    PKG_CONFIG_PATH="${PKG_CONFIG_PATH}:$(brew --prefix $pkg)/lib/pkgconfig"
done
export PKG_CONFIG_PATH="${PKG_CONFIG_PATH}"

export KERBEROS_LIBS="-lkrb5"
export KERBEROS_CFLAGS=" "
export EDIT_LIBS="-ledit"
export EDIT_CFLAGS=" "
```

### Default configuration

```sh
cat <<EOT >> $(phpenv prefix)/etc/conf.d/999-defaults.ini
memory_limit=-1
variables_order=EGPCS
opcache.enable_cli=1
error_reporting=-1
display_errors=1
display_startup_errors=1
date.timezone=UTC
EOT
```

### Install extensions

#### `zephir_parser`

```sh
git clone https://github.com/phalcon/php-zephir-parser.git /tmp/zephir-parser
cd /tmp/zephir-parser

phpenv local 7.4.0-zts-debug && phpenv rehash
phpize
./configure --with-php-config=$(phpenv which php-config) --enable-zephir-parser
make
make install

echo extension=zephir_parser.so > $(phpenv prefix)/etc/conf.d/zephir_parser.ini
```

#### `gmp`

```sh
cd $PHP_BUILD_TMPDIR/source/7.4.0/ext/gmp

phpenv local 7.4.0-zts-debug && phpenv rehash
phpize
./configure
make
make install

echo extension=gmp.so > "$(phpenv prefix)/etc/conf.d/gmp.ini"
```

### PECL extensions

```sh
phpenv local 7.4.0-zts-debug && phpenv rehash
phpenv pear-setup

export PHP_PEAR_PHP_BIN=$(phpenv which php)

printf "\n" | pecl install --force igbinary 1> /dev/null
printf "\n" | pecl install --force imagick 1> /dev/null
printf "\n" | pecl install --force yaml 1> /dev/null
printf "\n" | pecl install --force msgpack 1> /dev/null
printf "\n" | pecl install --force memcached 1> /dev/null
printf "\n" | pecl install --force psr 1> /dev/null
printf "\n" | pecl install --force apcu_bc 1> /dev/null
```
