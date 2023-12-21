# Настройка голого AstraLinux с репозиторием Smolensk для работы 1С-Битрикс и Битрикс24

<h2>Подготовка</h2>

```
apt-get install ca-certificates
apt-get install wget
apt-get install curl
apt-get install git
```
Создаем пользователя ```bitrix:bitrix``` с домашней директорией ```/home/bitrix/```, создаем папку ```/home/bitrix/www/```
```
mkdir /home/bitrix
useradd bitrix
usermod -aG www-data bitrix
```

<h2>Установка MariaDB (MySQL)</h2>

Получаем адрес свежего дистрибутива MariaDB. Например, получаем адрес tar.gz отсюда 
```https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.44-linux-glibc2.12-x86_64.tar.gz```



Распаковываем 

```tar -zxf mysql-5.7.44-linux-glibc2.12-x86_64.tar.gz```

Создаем пользователя, прокидываем симлинк
```
groupadd mysql
useradd -g mysql mysql
cd /usr/local
tar -zxvpf /path-to/mariadb-VERSION-OS.tar.gz
ln -s mariadb-VERSION-OS mysql
cd mysql
```
Теперь запускаем скрипт установки

```./scripts/mysql_install_db --user=mysql```

После установки ставим права
```
chown -R root . 
chown -R mysql data
```
Добавляем в ```~/.bashrc ```

```
PATH=${PATH}:/usr/local/mysql/bin:/usr/local/nginx
export PATH 
``` 

Копируем скрипт управления сервисом

```cp support-files/mysql.server /etc/init.d/mysql.server```

Пробуем тестово запустить

```./bin/mysqld_safe --user=mysql &```

Если видим консоль Mysql, значит все хорошо, выходим ```quit```

Устанавливаем конфиг my.cnf и папку mysql с конфигами из архива

Создаем БД (пароль меняем):
```
mysql
CREATE DATABASE `lks`;
CREATE USER 'lks_user' IDENTIFIED BY '1J1QlFMMl9k';
GRANT USAGE ON *.* TO 'lks_user'@'%' IDENTIFIED BY '1J1QlFMMl9k';
GRANT ALL PRIVILEGES ON `lks`.* TO 'lks_user'@'%';
GRANT USAGE ON *.* TO 'lks_user'@localhost IDENTIFIED BY '1J1QlFMMl9k';
GRANT ALL PRIVILEGES ON `lks`.* TO 'lks_user'@localhost;
FLUSH PRIVILEGES;
```

<h2>Установка PHP 7 и сервера PHP-FPM</h2>

```
apt-get install php7.0-fpm
```

Ставим необходимые модули php
```
apt-get -y --no-install-recommends install php-memcached
apt-get -y --no-install-recommends install php-memcache
apt-get -y --no-install-recommends install php-mysql
apt-get -y --no-install-recommends install php-intl
apt-get -y --no-install-recommends install php-interbase
apt-get -y --no-install-recommends install php-gd
apt-get -y --no-install-recommends install php-imagick
apt-get -y --no-install-recommends install php-mcrypt
apt-get -y --no-install-recommends install php-mbstring
apt-get -y --no-install-recommends install php-xml
apt-get -y --no-install-recommends install php-zip
apt-get -y --no-install-recommends install php-soap
apt-get -y --no-install-recommends install catdoc
```

Ставим права на логи
```
mkdir -p /var/log/php/
touch /var/log/php/error.log
chmod 775 /var/log/php/error.log
chown bitrix:bitrix /var/log/php/error.log
touch /var/log/php/opcache.log
chmod 775 /var/log/php/opcache.log
chown bitrix:bitrix /var/log/php/opcache.log
```

Заливаем конфиги в ```/etc/php/```

Собираем MSMTP (для отправки почты через внешний SMTP):
```
wget https://marlam.de/msmtp/releases/msmtp-1.8.0.tar.xz --no-check-certificate
tar -Jxf msmtp-1.8.0.tar.xz
cd msmtp-1.8.0
apt-get install pkg-config
apt-get install dh-autoreconf
apt-get install libgnutls30 libgnutls-openssl27 libgnutls-dane0 libgnutls28-dev libgnutlsxx28
autoreconf -i
./configure
make
sudo make install
```

Заливаем конфиг msmtprc из архива в ```~/.msmtprc``` и создаем симлинки ```/etc/msmtprc``` и ```/usr/local/etc/msmtprc```

Ставим права на конфиг и логи
```
chmod 0600 /etc/msmtprc 
chown bitrix:bitrix /etc/msmtprc
mkdir -p 	/var/log/msmtp/msmtp.log
touch /var/log/msmtp/msmtp.log
chmod 775 /var/log/msmtp/msmtp.log
chown bitrix:bitrix /var/log/msmtp/msmtp.log
```

Не забываем прописать в конфиге актуальные данные SMTP сервера!

Проверим почту 

```echo -e "test message" | /usr/local/bin/msmtp --debug -t -i name@site.ru```

В ответе ожидаем ```250 OK```

Управление сервисом:

```
systemctl status php7.0-fpm
systemctl start php7.0-fpm
systemctl stop php7.0-fpm
systemctl restart php7.0-fpm
```

<h2>Собираем nginx</h2>

Полезные инструкции (для справки)

```
https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/
https://dermanov.ru/exp/configure-push-and-pull-module-for-bitrix24/
https://onlinebd.ru/blog/1s-bitriks-nginx-php-fpm-kompozit
```

Устанавливаем нужные для сборки библиотеки 

```apt-get -y install build-essential zlib1g-dev libpcre3 libpcre3-dev libbz2-dev libssl-dev tar unzip```

Переходим в удобную папку, например, ```tmp``` и начинаем собирать nginx и библиотеки. Начинаем с PCRE:
```
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.42.tar.gz --no-check-certificate
tar -zxf pcre-8.42.tar.gz
cd pcre-8.42
./configure
make
sudo make install
```

Теперь zlib
```
wget http://zlib.net/zlib-1.2.11.tar.gz --no-check-certificate
tar -zxf zlib-1.2.11.tar.gz
cd zlib-1.2.11
./configure
make
sudo make install
```

OpenSSL:
```
wget http://www.openssl.org/source/openssl-1.0.2p.tar.gz --no-check-certificate
tar -zxf openssl-1.0.2p.tar.gz
cd openssl-1.0.2p
./Configure linux-x86_64 --prefix=/usr
make
sudo make install
```

Модуль для чатов (устаревший, можно пропустить, но не забыть убрать из сборки):
```
wget https://github.com/wandenberg/nginx-push-stream-module/archive/0.4.1.tar.gz --no-check-certificate
tar -zxf 0.4.1.tar.gz
```

Качаем nginx:
```
wget http://nginx.org/download/nginx-1.15.7.tar.gz --no-check-certificate
tar -zxf nginx-1.15.7.tar.gz 
```

Собираем и устанавливаем

```
./configure --sbin-path=/usr/local/nginx/nginx --conf-path=/usr/local/nginx/nginx.conf --pid-path=/usr/local/nginx/nginx.pid --add-module=../nginx-push-stream-module-0.4.1 --with-zlib=../zlib-1.2.11 --with-openssl=../openssl-1.0.2p --with-pcre=../pcre-8.42 --with-http_ssl_module --with-http_realip_module  --with-http_addition_module  --with-http_sub_module  --with-http_dav_module  --with-http_flv_module  --with-http_mp4_module  --with-http_gunzip_module  --with-http_gzip_static_module  --with-http_random_index_module  --with-http_secure_link_module  --with-http_stub_status_module  --with-http_auth_request_module  --with-http_v2_module  --with-mail  --with-mail_ssl_module  --with-file-aio  --with-ipv6 

make
make install
```
Копируем конфиги из файла в ```/etc/nginx```

создаем ```/lib/systemd/system/nginx.service```

```
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/local/nginx/nginx -t
ExecStart=/usr/local/nginx/nginx
ExecReload=/usr/local/nginx/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Пробрасываем симлинк с ```/etc/nginx/nginx.conf``` в папку nginx в ```/usr/local/nginx```, старый конфиг оттуда удалить.

Создать файлы логов ```/var/log/nginx/nginx_access.log``` и ```/var/log/nginx/nginx_error.log```, дать пользователя bitrix

Запускаем ```service nginx start```

Проверяем ```nginx -V``` и ```service nginx status```

<h3>Настройка memcached</h3>

Устанавливаем 
```
apt-get install memcached
```

Заливаем правильный `/etc/memcached.conf` из конфигов

Проверяем работу 
```
telnet localhost 11211
stats settings
quit
```

В битриксе настраиваем файлы (после внедрения битрикса)
В файле ```/bitrix/php_interface/dbconn.php```

```
define("BX_CACHE_TYPE", "memcache");
define("BX_CACHE_SID", $_SERVER["DOCUMENT_ROOT"]."#01");
define("BX_MEMCACHE_HOST", "localhost");
define("BX_MEMCACHE_PORT", "11211");
```

Создаем новый файл ```/bitrix/.settings_extra.php```

```
<?php
return array (
  'cache' => array(
     'value' => array (
        'type' => 'memcache',
        'memcache' => array(
            'host' => 'localhost',
            'port' => '11211'
        ),
        'sid' => $_SERVER["DOCUMENT_ROOT"]."#01"
     ),
  ),
);
```

Проверяем корректность установки с помощью инструментов "Проверка сайта" и "Монитор производительности". 

<h3>Прочее</h3>

Необходимо наличие флагов +x у всех папок от корня до скриптов в ```/home/bitrix/www```
Проверить можно так ```namei -om /home/bitrix/www/index.php```

Файлы ```.htaccess``` сервером php не обрабатываются, при необходимости пользуемся конвертером 
```http://winginx.com/ru/htaccess```

Необходимо открыть порты HTTP 8893, 8895 и HTTPS порт 8894. 

Необходимо настроить SSL сертификат на сервере или роутере. 

<h3>Установка 1C-Битрикс</h3>

Залить в папку ```/home/bitrix/www``` скрипт ```restore.php```, запустить в браузере, "загрузить архив с дальнего сайта", восстановить БД. 

<h3>Установка модуля чатов PushJS</h3>

Устанавливаем nodejs ```sudo apt install -y nodejs```

Устанавливаем redis ```sudo apt install redis-server```

Меняем у юзера redis группу на bitrix ```usermod -g bitrix redis```

Ставим права на папку c сокетом: ```chown -R redis:bitrix /var/run/redis/```

Добавляем конфиг redis из данного репозитория в папку ```/etc/redis/redis.conf```, старый удаляем. 

Настраиваем сервис redis в systemd ```vi /etc/systemd/system/redis.service```, прописываем
```
[Service]
Group=bitrix
```

Сохраняем изменения и перезапускаем
```
systemctl daemon-reload
systemctl enable redis && systemctl restart redis
```

Проверяем работу redis c помощью логов в ```/var/log/redis```, а также с помощью curl:
```
su bitrix
redis-cli -s /var/run/redis/redis-server.sock
> ping
```
Если видим ```PONG```, то redis работает

Ставим npm ```curl -L https://www.npmjs.com/install.sh | sh```

Теперь настраиваем запуск push-server. Создаем скрипт запуска ```/etc/init.d/push-server-multi``` - берем из данного репозитория. 
Размещаем конфиги push-server (все в данном репозитории). Основной конфиг ```push-server``` поместить в ```/etc/default/```, файлы шаблонов ```push-server-pub-__PORT__.json``` и ```push-server-sub-__PORT__.json``` поместить в ```/etc/push-server/```.

Распаковываем дистрибутив push-server:
```
mkdir /opt/push-server
sudo tar -xzf push-server.tar.gz -C /opt/push-server
sudo chown -R bitrix:bitrix /opt/push-server
chmod 755 `find /opt/push-server -type d`
```
и собираем его ```sudo npm install --production /opt/push-server/ 2>/dev/null```

Добавляем push-server в сервисы systemd (копируем из репозитория) ```/lib/systemd/system/push-server.service```
Активируем сервис
```
systemctl enable push-server.service
systemctl daemon-reload
```
Создаем папку для логов 
```
mkdir /var/log/push-server
chown -R bitrix:bitrix /var/log/push-server
```
Теперь пробуем инициировать модуль ```/etc/init.d/push-server-multi reset```
Результатом должны явиться шаблоны в папке ```/etc/push-server/```, а также логи в папке ```/var/log/push-server```, без ошибок. 
Если что-то пошло не так, нужно удалить шаблоны (кроме изначальных двух), логи и сделать ```killall node```, после устранения проблемы можно инициировать заново. 

Если все хорошо, стартуем ```systemctl start push-server```.

Переходим к настройке nginx. Прежде всего необходимо отключить старые конфиги чата. 
```
mv /etc/nginx/conf.d/push-im_settings.conf /etc/nginx/conf.d/push-im_settings.conf.bak
mv /etc/nginx/conf.d/push.conf /etc/nginx/conf.d/push.conf.bak
mv /etc/nginx/push-im_settings.conf /etc/nginx/push-im_settings.conf.bak
mv /etc/nginx/push-im_subscrider.conf /etc/nginx/push-im_subscrider.conf.bak
```
Заливаем новые конфиги из репозитория (папка ```/etc/nginx```). В конфиге ```ssl.conf``` необходимо прописать пути к самоподписанному сертификату сервера, приватному ключу и файлу dhparam. Тестируем nginx ```nginx -t``` и если ошибок нет (может только ругаться на сертификат), то рестартуем nginx ```service nginx restart```

Подключаем к битриксу. Извлекаем сгенерированный ключ безопасности ```grep SECURITY_KEY /etc/default/push-server```, он идет сразу после ```SECURITY_KEY=```
Редактируем файл ```/home/bitrix/www/bitrix/.settings_extra.php```, заменяем из репозитория и подставляем свой ключ безопасности в ветку ```signature_key```.
В настройках модуля Push&Pull необходимо включить поддержку Bitrix Push Server (последний радиопин).
После этого нужно провести проверку конфигурации сайта и работы портала ```/bitrix/admin/site_checker.php?lang=ru```

