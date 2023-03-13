# Nginx AWS Authentication module

Генерирует Authorization токен для v4

## Установка

| Тестрировалось на Ubuntu 20.04, nginx 1.17.10

### Необходимые пакеты

```bash
apt update

apt install git \
    make \
    gcc \
    autoconf \
    wget \
    libpcre3-dev \
    libpcre++-dev \
    zlib1g-dev \
    libxml2-dev \
    libxslt-dev \
    libgd-dev \
    libgeoip-dev

apt install nginx
```

### Nginx configuration

```bash
nginx -V
```

Пример вывода:

```bash
nginx version: nginx/1.17.10 (Ubuntu)
built with OpenSSL 1.1.1f  31 Mar 2020
TLS SNI support enabled
configure arguments: --with-cc-opt='-g -O2 -fdebug-prefix-map=/build/nginx-Pmk9_C/nginx-1.17.10=. -fstack-protector-strong -Wformat -Werror=format-security -fPIC -Wdate-time -D_FORTIFY_SOURCE=2' --with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now -fPIC' --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid --modules-path=/usr/lib/nginx/modules --http-client-body-temp-path=/var/lib/nginx/body --http-fastcgi-temp-path=/var/lib/nginx/fastcgi --http-proxy-temp-path=/var/lib/nginx/proxy --http-scgi-temp-path=/var/lib/nginx/scgi --http-uwsgi-temp-path=/var/lib/nginx/uwsgi --with-debug --with-compat --with-pcre-jit --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --with-http_v2_module --with-http_dav_module --with-http_slice_module --with-threads --with-http_addition_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_image_filter_module=dynamic --with-http_sub_module --with-http_xslt_module=dynamic --with-stream=dynamic --with-stream_ssl_module --with-mail=dynamic --with-mail_ssl_module
```

### Подготавливаем модуль

* Скачиваем исходный код nginx для нашей версии (1.17.10), например, в папку ```/tmp/nginx```
* Распаковываем его
* Туда же скачиваем репозиторий

Переходим туда

```bash
cd /tmp/nginx
ls
```

Результат вывода ls:

```bash
nginx-1.17.10.tar.gz
nginx-1.17.10/
nginx-aws-auth-module/
```

### Собираем nginx с нашим модулем

```bash
cd /tmp/nginx/nginx-1.17.10

./configure ${configure_arguments} --add-dynamic-module=../nginx-aws-auth-module

make modules
```

```configure_arguments``` это аргументы из вывода ```nginx -V```

### Подключаем модуль

#### Копируем его в папку с модулями

```bash
cp /tmp/nginx/nginx-1.17.10/objs/ngx_http_aws_auth_module.so /usr/share/nginx/modules
```

#### Делаем конфиг

```bash
touch /etc/nginx/modules/ngx_http_aws_auth_module.conf
```

И вставляем в него:

```conf
load_module modules/ngx_http_aws_auth_module.so;
```

Чтобы это работало в файле ```/etc/nginx/nginx.conf``` должна быть строка:

```conf
include /etc/nginx/modules/*.conf;
```

### Проверяем

```bash
nginx -t
```

Если ошибок нету, то все хорошо - можно сразу переходит к настройке конфига

А если видно следующее, то нет:

```bash
nginx: [emerg] module "/usr/share/nginx/modules/ngx_http_aws_auth_module.so" is not binary compatible in /etc/nginx/modules/ngx_http_aws_auth_module.conf:1
```

#### Смотрим строку сингнатур бинарного файла nginx

```bash
strings /usr/sbin/nginx | fgrep '8,4,8'
```

Пример вывода:

```bash
8,4,8,0011111111010111001111111111111111
```

#### Смотрим строку сингнатур бинарного файла собраного модуля

```bash
strings /usr/share/nginx/modules/ngx_http_aws_auth_module.so | fgrep '8,4,8'
```

Пример вывода

```bash
8,4,8,0011111111010111001111011111111111
```

#### Видим что в этих строках отличается 12-й символ с конца

Открываем файл

```bash
pico /tmp/nginx/nginx-1.17.10/src/core/ngx_module.h
```

И в нем смотрим 12-ю сигнатуру с конца, в нашем случае это

```cpp
#if (NGX_PCRE)
#define NGX_MODULE_SIGNATURE_23  "1"
#else
#define NGX_MODULE_SIGNATURE_23  "0"
#endif
```

Сделать с этим можно следующее

* Переустановить libpcre++-dev
* Открыть файл конфига в модуле и дописать там следующее

```bash
cd /tmp/nginx/nginx-aws-auth-module

pico config
```

```txt
ngx_addon_name=ngx_http_aws_auth

have=NGX_PCRE . auto/have
```

После этого заново все пересобрать

## Nginx.conf

Пример:

```conf
http {
  aws_auth $aws_token {
    access_key "<access_key>";
    secret_key "<secret_key>";
    service "s3";
    region "<REGION>";
  }

  location ~* ^/uploads/(.*) {
    resolver 8.8.4.4 8.8.8.8 valid=300s;
    resolver_timeout 10s;

    set $aws_bucket "<BUCKET>";
    set $aws_endpoint "<ENDPOINT>";
    set $random_hex "<RANDOM HEX>"

    rewrite ^ $request_uri; # get original URI
    rewrite ^/uploads/(.*) $1  break;  # drop /uploads/, put /*

    proxy_set_header X-Amz-Date $aws_auth_date;
    proxy_set_header Authorization $aws_token;
    proxy_set_header X-Amz-Content-SHA256 $random_hex;

    proxy_pass https://$aws_endpoint/$aws_bucket/$uri;
  }
}
```

## Дополнительные ссылки

* [Добавление модулей nginx в Linux](https://firstwiki.ru/index.php/%D0%94%D0%BE%D0%B1%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5_%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D0%B5%D0%B9_nginx_%D0%B2_Linux_(Debian/Ubuntu/CentOS))
* [Оригинальный репозиторий](https://github.com/kaltura/nginx-aws-auth-module)
* [nginx](https://nginx.org/download/)
  
