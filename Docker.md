# Установка Пассворк с помощью Docker

### Установка Docker
Скачайте и установите Docker CE (https://docs.docker.com/engine/installation/).  
Обновите систему:

```
sudo –i
apt-get update
apt-get upgrade 
```

### Установите Git

```
apt-get install git 
```

### Скачайте файлы управления и шаблонные конфиг-файлы
Создайте директорию /server и сколнируйте файлы:

```
mkdir /server
git clone https://github.com/passwork-me/docker.git /server
```

### Скачайте исходные файлы Пассворк
Удалите файл .gitkeep:

```
rm /server/sites/prod/.gitkeep
```

Сколнируйте репозиторий. Укажите логин и пароль, которые вам сообщили менеджеры Пассворк. 

```
git clone http://passwork.download/passwork/passwork.git /server/sites/prod
cd /server/sites/prod/
git checkout php7
```

### Создайте config.ini

```
cp /server/sites/prod/app/config/config.example.ini /server/sites/prod/app/config/config.ini
```

### Запустите контейнер Nginx

```
docker run -d --name=nginx \
--restart unless-stopped \
-p 80:80 -p 443:443  \
-v /server/conf/nginx:/server/conf/nginx \
-v /server/conf/php:/server/conf/php \
-v /server/conf/ssl:/server/conf/ssl \
-v /server/conf/postfix:/server/conf/postfix \
-v /server/log/nginx:/server/log/nginx \
-v /server/log/php:/server/log/php \
-v /server/log/syslog:/server/log/syslog \
-v /server/sites/:/server/sites/ \
passwork/nginx-php7 \
/bin/bash -c "/server/run"
```

### Запустите контейнер MongoDB

```
docker run -d --name=db \
--restart unless-stopped \
-p 27017:27017 \
-v /server/conf/mongo:/server/conf/ \
-v /server/log/mongo:/server/log/ \
-v /server/data/mongo/:/server/data \
passwork/mongo \
/server/run
```

Обратите внимание, что порт 27017 по умолчанию открыт без аутентификации.
Настройте ваш файрволл, чтобы блокировать внешние подключения к порту 27017.


Проверьте, что контейнере запустились и работают:

```
docker ps
```

### Создание базы данных
Скопируйте бекап базы данных:

```
cp -r /server/sites/prod/dump/ /server/data/mongo/dump
```

Выполните *mongorestore* утилиту, чтобы развернуть бекап:

```
docker exec -i db mongorestore /server/data/dump
```

### Настройка config.ini 
Определите какой IP Docker установил для контейнра с MongoDB

```
docker inspect db | grep IPAddress
```

Вернет:

```
…
"IPAddress": "172.17.0.3",
…
```

Найдите секцию [mongo] и пропишите этот IP адрес. 

```
mcedit /server/sites/prod/app/config/config.ini
```

```
[mongo]
connectionString = mongodb://172.17.0.3:27017
dbname = pwbox
useCreds = false
username =
password =
```

### Установка лицензии.

Распакуйте архив с ключами для регистрации и переместите файлы .lic и reginfo.json (или reginfo.php) в директорию `/server/sites/prod/app/keys/`.
Ключи так же можно загрузить через веб-интерфейс (Меню → Информация о компании)

## Готово
Откройте ваш Пассворк в браузере и вы увидите окно авторизации.
Используйте встроенную учетную запись:

```
Login: admin@passwork.me
Password: DemoDemo
```

## О Docker-образах
Docker-образы устроены таким образом, что все важные данные вынесены в общие с хостовой машиной папки. Поэтому вы можете смело останавливать и удалять и создавать новые контейнеры.
Конфигурационные файлы так же хранятся в общих папках (т.е. не в контейнере), поэтому вы можете провести любую настройку Nginx, PHP и MongoDB без внесения изменений непосредственно к образы или контейнеры. Просто внесите изменения в конфиг файлы и перезапустите контейнер или сервисы.

Если вам необходимо внести изменения в образ, то войдите в контейнер, внестите изменения и затем необходимо закоммитить в ваш образ командой

```
docker commit <container id> you-name/image-name
```

Более подробно описано в официальной документации Docker. 



## Полезные команды
Скопируйте утилиту *dexec* в /usr/bin/:

```
cp /server/dexec /usr/bin/dexec
```

Вход в контейнер:

```
docker exec -it <container> bash
```

Восстановление прав для файлов сайта (требуется после обновления):

```
/server/docker-nginx-permissions nginx
```

Перезагрузка Nginx без остановки:

```
/server/docker-nginx-reload nginx
/server/docker-php-reload nginx
```

Контейнеры запущены с опицией *autostart*.
Это означает, что Docker автоматически перезапустит контейнер, если он по каким-либо причинам остановится.
Поэтому, если вам необходимо остановить контейнер, сперва отключите *autostart*:

```
/server/docker-norestart <container>
```

Включить *autostart* обратно:

```
/server/docker-autorestart <container>
```

Без опции *autostart* вы можете остановить контейнеры:

```
/server/docker-nginx-stop nginx
/server/docker-mongo-stop db
```

Обратите внимание, что если опция *autostart* включена, то эти команды перезапустят Nginx и MongoDB,

Принудительная остановка контейнеров:

```
docker stop <container>
```

Используйте ее в крайних случаях, так как это может повлечь порчу данных.

## Структура файлов
Файлы конфигураций:

```
/server/conf/
```

Данные (база данных):

```
/server/data/
```

Логи:

```
/server/log/
```

Сайты:

```
/server/sites/
```

## Пример: Как изменить конфигурацию Nginx или PHP
Отредактируйте файлы:

```
mcedit /server/conf/nginx/prod.site
mcedit /server/conf/nginx/nginx.conf
mcedit /server/conf/php/php.ini
```

Перезапустите nginx и php-fpm:

```
/server/docker-nginx-reload nginx
/server/docker-php-reload nginx
```

## Настройка почты
Nginx контейнер использует Postfix для отправки почты. 
Все конфигурационные файлы вы можете найти здесь:

```
/server/conf/postfix/
```

Отредактируйте их под свои нужды. 
Перезапустите Postfix, чтобы изменения вступили в силу.

```
docker exec -i nginx service postfix reload
```

## Пример настройки Postfix
Откройте конфигурационный файл `/server/conf/postfix/main.cf`.

```
mcedit /server/conf/postfix/main.cf
```


Убедитесь в том, что параметр myhostname совпадает с полным доменным именем вашего сервера:

```
myhostname = passwork
```


### Настройка имен и паролей SMTP.

Откройте или создайте файл `/server/conf/postfix/sasl_passwd`.

```
mcedit /server/conf/postfix/sasl_passwd
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
docker exec -it nginx postmap /etc/postfix/sasl_passwd
```


После успешного выполнения команды в директории /server/conf/postfix должен появиться новый файл sasl_passwd.db.


### Защита файла с паролями и хэш-файла.

Файлы /server/conf/postfix/sasl_passwd и /server/conf/postfix/sasl_passwd.db, созданные в предыдущих шагах, содержат ваши учетные данные SMTP в виде простого текста.
По соображениям безопасности вы должны изменить права доступа к ним, так чтобы только пользователь root мог читать и записывать в файл.

Выполните следующие команды, чтобы изменить владельца файлов на root и обновить права доступа для файлов:

```
chown root:root /server/conf/postfix/sasl_passwd /server/conf/postfix/sasl_passwd.db
chmod 0600 /server/conf/postfix/sasl_passwd /server/conf/postfix/sasl_passwd.db
```


### Конфигурация релей сервера

Откройте файл `/server/conf/postfix/main.cf`.

```
mcedit /server/conf/postfix/main.cf
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
docker exec -i nginx service postfix reload
```
