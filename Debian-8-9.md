# Установка менеджера паролей в Debian 8, 9

**1. Получение прав root и создание локальной базы данных пакетов.**

```
su
cd ~
apt-get update
```


Измените имя сервера на "passwork".

```
hostnamectl set-hostname passwork
```


Установите сервисы для автоматической настройки и обнаружения в локальной сети.
```
apt-get install -y avahi-daemon libnss-mdns
```


Измените значение параметра `AVAHI_DAEMON_DETECT_LOCAL` с 1 на 0.

```
nano /etc/default/avahi-daemon
```


Приведите содержимое файла к виду:

```
AVAHI_DAEMON_DETECT_LOCAL = 0
```


Перезапустите avahi-daemon:

```
service avahi-daemon restart
```


**2. Установка Git и Apache2.**

```
apt-get install -y git apache2
```


**3. Установка MongoDB.**

Импортируйте открытый ключ, используемый системой управления пакетами.

```
apt-get install -y dirmngr
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2930ADAE8CAF5059EE73BB4B58712A2291FA4AD5
```


Создайте файл /etc/apt/sources.list.d/mongodb-org-3.6.list

```
echo "deb http://repo.mongodb.org/apt/debian jessie/mongodb-org/3.6 main" | tee /etc/apt/sources.list.d/mongodb-org-3.6.list
echo "deb http://ftp.debian.org/debian jessie-backports main" >> /etc/apt/sources.list
```


Обновите локальную базу данных пакетов.

```
apt-get update
```


Установите последнюю стабильную версию MongoDB.

```
apt-get install -y mongodb-org
```


Запустите службу MongoDB.

```
service mongod start
```


Включите автозапуск службы mongod при загрузке системы.

```
systemctl enable mongod.service
```


**4. Установка PHP5.6.**

```
apt-get install -y apt-transport-https lsb-release ca-certificates
```


Получите gpg ключ:

```
wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
```


Добавьте новый репозиторий в список источников:

```
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
```


**Установка PHP и дополнительных расширений.**

```
apt-get update
apt-get install -y php5.6 php5.6-json php5.6-mcrypt php5.6-dev php5.6-ldap php5.6-xml php5.6-bcmath php5.6-mbstring
```


**5. Установка PHP Mongo драйвера.**

```
apt-get install -y pkg-config
pecl install mongo
```


В процессе установки введите ответ “no” на запрос установщика.

```
echo "extension=mongo.so" | tee /etc/php/5.6/apache2/conf.d/20-mongo.ini
```


**6. Установка Phalcon PHP фреймворка.**

```
git clone --depth=1 "git://github.com/phalcon/cphalcon.git"
cd cphalcon/build
./install
echo "extension=phalcon.so" | tee /etc/php/5.6/apache2/conf.d/20-phalcon.ini
service apache2 restart
```


**7. Загрузка и установка Passwork.**

Клонируйте репозиторий используя ваш логин и пароль.

```
cd /var/www
git init
git remote add origin https://passwork.download/passwork/passwork.git
git pull origin master
```


Используйте имя пользователя и пароль для получения доступа к репозиторию.

Создайте кофигурационный файл и установите права доступа на папки и файлы.

```
cp /var/www/app/config/config.example.ini /var/www/app/config/config.ini
find /var/www/ -type d -exec chmod 755 {} \;
find /var/www/ -type f -exec chmod 644 {} \;
chown -R www-data:www-data /var/www/
```


Восстановите базу данных MongoDB.

```
mongorestore /var/www/dump/
```


**Конфигурация Apache2.**

Откройте конфигурационный файл Apache.

```
nano /etc/apache2/sites-enabled/000-default.conf
```


Приведите содержимое файла к следующему виду:

```
<VirtualHost *:80>
    #ServerName .....
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/public
    <Directory /var/www/public>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```


Включите rewrite модуль и перезапустите Apache.

```
a2enmod rewrite
service apache2 restart
```


**Установка лицензии.**

Распакуйте архив с ключами для регистрации и переместите файлы "demo.openssl.lic" и "reginfo.php" в директорию "/var/www/app/keys/".


**Установка завершена.**

Откройте [http://passwork.local](http://passwork.local) для доступа к вебсайту.


**Используйте учетную запись по умолчанию для входа в систему:**

логин: `admin@passwork.me`

пароль: `DemoDemo`


**8. Создание SSL сертификата.**

Активируйте SSL модуль Apache.

```
a2enmod ssl
```


Активируйте стандартную конфигурацию SSL.

```
a2ensite default-ssl
```


Перезапустите Apache для того, чтобы изменения вступили в силу.

```
service apache2 restart
```


Создайте новую директорию для хранения закрытого ключа и сертификата.

```
mkdir /etc/apache2/ssl
```


Создайте новый сертификат и закрытый ключ для его защиты.

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/apache2/ssl/apache.key -out /etc/apache2/ssl/apache.crt
```


Вызов этой команды приведет к серии запросов.

* Common Name: Укажите IP-адрес вашего сервера или имя хоста. Это поле имеет значение, так как ваш сертификат должен соответствовать домену (или IP-адресу) для вашего веб-сайта.

* Заполните все остальные поля по своему усмотрению.

Ниже приведены примеры ответов.

```
Interactive
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
——
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:Example
Locality Name (eg, city) []:Example
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Example Inc
Organizational Unit Name (eg, section) []:Example Dept
Common Name (e.g. server FQDN or YOUR name) []:passwork.local
Email Address []:test@passwork.local
```


Установите права доступа для защиты закрытого ключа и сертификата.

```
chmod 600 /etc/apache2/ssl/*
```


Ваш сертификат и закрытый ключ, который его защищает, готовы для использования с Apache.


**9. Настройка Apache для использования SSL-соединения.**

Откройте кофигурационный файл SSL в текстовом редакторе.

```
nano /etc/apache2/sites-enabled/default-ssl.conf
```


Найдите раздел, начинающийся с `<VirtualHost _default_:443>` и внесите следующие изменения.

* Добавьте строку с директивой имени сервера под строкой `ServerAdmin`. Это может быть ваше доменное имя или IP-адрес:

```
ServerAdmin webmaster@localhost
ServerName passwork.local:443
```


* Добавьте директиву “Directory” после “ServerName”.

```
    <Directory /var/www/public>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>
```


* Найдите следующие две строки и обновите пути к файлам, соответствующие местоположению сертификата и ключа, которые мы создали ранее. Если вы приобрели сертификат или создали свой сертификат в другом месте, убедитесь, что путь к файлу соответствует фактическому местоположению вашего сертификата и ключа:

```
SSLCertificateFile /etc/apache2/ssl/apache.crt
SSLCertificateKeyFile /etc/apache2/ssl/apache.key
```


После внесения изменений проверьте, что ваш файл конфигурации виртуального хоста соответствует примеру ниже.

```
<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
        ServerAdmin webmaster@localhost
        ServerName passwork.local:443
        DocumentRoot /var/www/public
        <Directory /var/www/public>
            Options Indexes FollowSymLinks MultiViews
            AllowOverride All
            Order allow,deny
            allow from all
        </Directory>
        SSLEngine on
        SSLCertificateFile /etc/apache2/ssl/apache.crt
        SSLCertificateKeyFile /etc/apache2/ssl/apache.key
    </VirtualHost>
</IfModule>
```


Перезапустите Apache, чтобы изменения вступили в силу.

```
service apache2 restart
```


Проверьте SSL-соединение, откройте ссылку [https://passwork.local](https://passwork.local).


**10. Установка Postfix.**

Установите Postfix при помощи следующей команды:

```
apt-get install -y postfix
```


Во время установки выберите тип конфигурации Postfix - "Internet Site".

![alt text](./images/Debian8_9_01.png)

Введите полное имя домена, passwork.

![alt text](./images/Debian8_9_02.png)

После завершения установки откройте конфигурационный файл /etc/postfix/main.cf.

```
nano /etc/postfix/main.cf
```


Убедитесь в том, что параметр myhostname совпадает с полным доменным именем вашего сервера:

```
myhostname = passwork
```


**Настройка имен и паролей SMTP.**

Откройте или создайте файл /etc/postfix/sasl_passwd.

```
nano /etc/postfix/sasl_passwd
```


Добавьте SMTP хост, имя пользователя и пароль должны быть записаны в следующем формате:

```
[mail.isp.example] username:password
```


Если вы хотите использовать нестандартный TCP-порт (например, 587), используйте следующий формат:

```
[mail.isp.example]:587 username:password
```


**для Gmail запись будет выглядеть следующим образом:**
```
[smtp.gmail.com]:587 username:password
```


Создайте хэшированную базу данных для Postfix, выполните команду postmap:

```
postmap /etc/postfix/sasl_passwd
```


После успешного выполнения команды в директории /etc/postfix/directory должен появиться новый файл sasl_passwd.db.


**Защита файла с паролями и хэш-файла.**

Файлы /etc/postfix/sasl_passwd и /etc/postfix/sasl_passwd.db, созданные в предыдущих шагах, содержат ваши учетные данные SMTP в виде простого текста.
По соображениям безопасности вы должны изменить права доступа к ним, так чтобы только пользователь root мог читать и записывать в файл.

Выполните следующие команды, чтобы изменить владельца файлов на root и обновить права доступа для файлов:

```
chown root:root /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
chmod 0600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
```


**Конфигурация релей сервера.**

Откройте файл /etc/postfix/main.cf.

```
nano /etc/postfix/main.cf
```


Измените параметр relayhost, на свой внешний SMTP релей. Если в файле sasl_passwd был указан нестандартный TCP-порт, то вы должны использовать тот же порт при настройке параметра relayhost.

Укажите SMTP релей:

```
relayhost = [mail.isp.example]:587
```


**для Gmail запись будет выглядеть следующим образом:**

```
relayhost = [smtp.gmail.com]:587
```


В конце файла добавьте следующие параметры для включения аутентификации:

```
# enable SASL authentication
smtp_sasl_auth_enable = yes
# disallow methods that allow anonymous authentication.
smtp_sasl_security_options = noanonymous
# where to find sasl_passwd
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
# Enable STARTTLS encryption
smtp_use_tls = yes
# where to find CA certificates
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
```


Сохраните изменения.

Перезапустите Postfix:

```
service postfix restart
```