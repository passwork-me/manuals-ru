# Установка менеджера паролей в CentOS 7

**1. Получение прав root и создание локальной базы данных пакетов.**

```
su
cd ~
yum makecache
```


Измените имя сервера на "passwork".

```
hostnamectl set-hostname passwork
/etc/init.d/network restart
```


**2. Установка Git и Apache2, настройка файрвола.**

```
yum -y install git httpd avahi
systemctl start httpd
systemctl enable httpd
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=5353/udp
firewall-cmd --reload
systemctl restart avahi-daemon
```


**3. Установка MongoDB.**

Настройка системы управления пакетами (yum).

Создайте файл /etc/yum.repos.d/mongodb-org-3.6.repo для того, чтобы установить MongoDB с использованием менеджера пакетов yum.

```
yum -y install nano
nano /etc/yum.repos.d/mongodb-org-3.6.repo
```


Приведите файл к следующему виду:

```
[mongodb-org-3.6]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.6/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.6.asc
```


Для установки последней стабильной версии MongoDB выполните следующую комманду:

```
yum -y install mongodb-org
```


Измените режим работы SELinux на permissive. Откройте файл /etc/selinux/config и измените значение параметра `SELINUX` на permissive.

```
nano /etc/selinux/config
```


Приведите файл к следующему виду:

```
SELINUX=permissive
```


Для того, чтобы внесенные изменения вступили в силу необходимо перезагрузить систему.

Запустите службу MongoDB.

```
su
cd ~
service mongod start
```


Включите автозапуск службы mongod при загрузке системы.

```
systemctl enable mongod.service
```


**4. Установка PHP5.6.**

**Установка пакета конфигурации Remi репозитория.**

```
yum -y install wget yum-utils
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
rpm -Uvh remi-release-7*.rpm epel-release-latest-7*.rpm
yum-config-manager --enable remi-php56
```


**Установка PHP и дополнительных расширений.**

```
yum -y install php php-json php-mcrypt php-ldap php-xml php-bcmath php-mbstring
```


**5. Установка PHP Mongo драйвера.**

```
yum -y install gcc php-pear php-devel openssl-devel
pecl install mongo
```


В процессе установки введите ответ “no” на запрос установщика.

```
echo "extension=mongo.so" | tee /etc/php.d/20-mongo.ini
systemctl restart httpd
```


**6. Установка Phalcon PHP фреймворка.**

```
yum -y install php-mysql libtool pcre-devel
git clone --depth=1 "git://github.com/phalcon/cphalcon.git"
cd cphalcon/build
./install
echo "extension=phalcon.so" | tee /etc/php.d/50-phalcon.ini
systemctl restart httpd
```


**7. Загрузка и установка Passwork.**

Клонируйте репозиторий используя ваш логин и пароль.

```
cd /var/www
git init
git remote add origin http://get.passwork.pro:81/passwork/passwork.git
git pull origin master
```


Используйте имя пользователя и пароль для получения доступа к репозиторию.

Создайте кофигурационный файл и установите права доступа на папки и файлы.

```
cp /var/www/app/config/config.example.ini /var/www/app/config/config.ini
find /var/www/ -type d -exec chmod 755 {} \;
find /var/www/ -type f -exec chmod 644 {} \;
chown -R apache:apache /var/www/
```


Восстановите базу данных MongoDB.

```
mongorestore /var/www/dump/
```


**Конфигурация Apache2.**

Создайте конфигурационный файл для non-ssl соединения.

```
nano /etc/httpd/conf.d/non-ssl.conf
```


Приведите содержимое файла к следующему виду:

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/public
    <Directory /var/www/public>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>
    ErrorLog logs/error_log
    TransferLog logs/access_log
    LogLevel warn
</VirtualHost>
```


Перезапустите Apache.

```
systemctl restart httpd
```


**Установка лицензии.**

Распакуйте архив с ключами для регистрации и переместите файлы "demo.openssl.lic" и "reginfo.php" в директорию "/var/www/app/keys/".


**Установка завершена.**

Откройте [http://passwork.local](http://passwork.local) для доступа к вебсайту.


**Используйте учетную запись по умолчанию для входа в систему:**

логин: `admin@passwork.me`

пароль: `DemoDemo`


**8. Создание SSL сертификата.**

Установите SSL модуль для Apache.

```
yum -y install mod_ssl
```


Создайте новую директорию для хранения закрытого ключа (директория /etc/ssl/certs доступна по умолчанию для хранения файла сертификата):

```
mkdir /etc/ssl/private
```


Установите правильные разрешения на директорию:

```
chmod 700 /etc/ssl/private
```


Создайте новый сертификат и закрытый ключ для его защиты.

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
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


Создайте группу Диффи-Хеллмана.

```
openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```


Добавьте содержимое сгенерированного файла в конец самоподписанного сертификата.

```
cat /etc/ssl/certs/dhparam.pem | tee -a /etc/ssl/certs/apache-selfsigned.crt
```


**9. Настройка Apache для использования SSL-соединения.**

Откройте кофигурационный файл SSL в текстовом редакторе.

```
nano /etc/httpd/conf.d/ssl.conf
```


Найдите раздел, начинающийся с `<VirtualHost _default_:443>` и внесите следующие изменения.

* Раскомментируйте строку `DocumentRoot` и измените путь в кавычках на путь к корневому каталогу вашего сайта. По умолчанию это будет /var/www/html.

* Затем, раскомментируйте строку `ServerName` и замените `www.example.com` на IP-адрес или домен сервера (в зависимости от того, что вы указали в качестве `Common Name` в своем сертификате):

```
DocumentRoot /var/www/public
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


* Затем найдите строки SSLProtocol и SSLCipherSuite и либо удалите их, либо закомментируйте. Новая конфигурация, которую мы добавим ниже имеет более безопасные параметры, чем настройки Apache по умолчанию в CentOS:

```
# SSLProtocol all -SSLv2
# SSLCipherSuite HIGH:MEDIUM:!aNULL:!MD5:!SEED:!IDEA
```


Найдите строки SSLCertificateFile и SSLCertificateKeyFile и приведите их к следующему виду:

```
SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
```


После внесения изменений проверьте, что ваш файл конфигурации виртуального хоста соответствует примеру ниже.

```
<VirtualHost _default_:443>
    DocumentRoot /var/www/public
    ServerName passwork.local:443
    <Directory /var/www/public>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
</VirtualHost>
```


Мы закончили с изменениями в блоке VirtualHost. Сохраните изменения (Ctr+O) и выйдите (Ctr+X).

Перезапустите Apache, чтобы изменения вступили в силу.

```
systemctl restart httpd
```


Проверьте SSL-соединение, откройте ссылку [https://passwork.local](https://passwork.local).


**10. Установка Postfix.**

Установите Postfix при помощи следующей команды:

```
yum -y install postfix cyrus-sasl-plain mailx
systemctl restart postfix
```


Включите автозапуск службы Postfix при загрузке системы.

```
systemctl enable postfix
```


Откройте файл конфигурации /etc/postfix/main.cf

```
nano /etc/postfix/main.cf
```


и добавьте следующие строки в конец файла.

```
myhostname = passwork
relayhost = [smtp.gmail.com]:587
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.crt
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
```


Сохраните изменения файла main.cf и закройте текстовый редактор.


**Конфигурация SASL аутентификации.**

Для аутентификации необходимо добавить учетные данные Gmail. Создайте файл /etc/postfix/sasl_passwd file

``` 
nano /etc/postfix/sasl_passwd
```


и добавьте следующую строку:

```
[smtp.gmail.com]:587 username:password
```


Значения имени пользователя и пароля должны быть заменены действительными учетными данными Gmail. Теперь файл sasl_passwd можно сохранить и закрыть.

Cоздаем таблицу поиска Postfix, выполнив следующую команду.

```
postmap /etc/postfix/sasl_passwd
```


Доступ к файлам sasl_passwd должен быть ограничен. Выставляем корректные права доступа.

```
chown root:postfix /etc/postfix/sasl_passwd*
chmod 640 /etc/postfix/sasl_passwd*
```


Перезагружаем конфигурацию Postfix.

```
systemctl reload postfix
```