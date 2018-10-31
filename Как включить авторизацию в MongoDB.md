# Как включить авторизацию в MongoDB

По умолчанию коробочная версия Пассворка не использует доступ к базе MongoDB с авторизацией. Вы можете включить и настроить авторизацию в MongoDB для повышения безопасности.

### Создание пользователя в MongoDB
Откройте командую строку и выполните:   
```
mongo
```
для подключения к вашей MongoDB.  
Создайте пользователя при помощи команд:
```
use admin;
db.createUser(
{
	user: "adminuser",
	pwd: "password",
   roles: [ 
   	{ role: "userAdminAnyDatabase", db: "admin" },
		{ role: "dbOwner", db: "admin" }, 
		{ role: "dbOwner", db: "pwbox" },
		{ role: "dbOwner", db: "pwbox-cache" } ]
})  

```

`user` — логин пользователя
`pwd` — пароль пользователя  
Не используйте в пароле пользователя символы `@`, `:`, `#` т.к. это может привести к сбоям в работе приложения. 

### Включение режима авторизации
#### Найдите конфигурационный файл MongoDB
В зависимости от вашей операционной системы и версии базы конфигурационный файл может называться: 
* mongod.conf  
* mongod.cfg  
* mongodb.conf  

В Linux этот файл обычно находится в директории `/etc/`  
В Windows — в папке, в которой установлена база данных.   
Вы можете найти путь до конфигурационного файла с помощью следующих команд   

```
mongo
use admin
db.adminCommand( { getParameter : '*' } )
```

#### Включите режим авторизации  
Откройте файл конфигурации и добавьте в него строку:   
```
security.authorization: enabled
```
Сохраните файла и перезапустите MongoDB.
  
### Настройте Пассворк для работы с новым режимом  
Найдите и откройте файл `<пассворк>/app/config.ini`.     
Найдите параметр `connectionString`:
```
[mongo]
connectionString = mongodb://localhost:27017
dbname = pwbox
```
и замените на:  
  
```
connectionString = mongodb://adminuser:password@localhost:27017
dbname = pwbox
```
Сохраните файл и обновите страницу в браузере с Пассворком.
    
## Диагностирование проблем  
### Обнаружение проблем  
•	после описанных действий Пассворк престал работать корректно  
•	на странице отображается строка `{"response":false}`  
  
Проверьте лог файлы Пассворка, которые находятся в `<пассворк>/app/logs/`.    
Если видите такие строки:  
```
Failed to connect to: localhost:27017: Authentication failed on database 'admin' with username adminuser: auth failed
#0 C:\inetpub\wwwroot\app\config\services.php(484): MongoClient->__construct('mongodb://usera...')
```
То скорее всего проблема заключается в том, что у вас установлено устаревшее PHP-расширение `mongo`. Откройте в браузере страницу `http://ваш_пассворк/check.php` — версия `mongo` должна быть 1.6.

Попробуйте выполнить следующие действия.   
В конфигурационном файле MongoDB закоментируйте или удалите строку:
```
security.authorization: enabled
```
Сохраните файл и перезагрузите MongoDB.
Откройте командую строку:
```
mongo
use admin
  
var schema = db.system.version.findOne({"_id" : "authSchema"});
schema.currentVersion = 3;
db.system.version.save(schema);
```
В конфигурационном файле MongoDB верните строку:  
```
security.authorization: enabled
```

И снова перезапустите MongoDB.   

Откройте Пассворк в браузере, проверьте его работу. 
## Техническая поддержка
Если вы по-прежнему наблюдаете проблемы, то пришлите на <mail@passwork.ru> следующую информацию: 
* скриншот страницы `http://ваш_пассворк/check.php`  
* скриншот конфиг файла Пассворка `<пассворк>/app/config.ini`.  
* последний лог файл Пассворка из `<пассворк>/app/logs/`  
* скриншот конфиг-файла MongoDB  
* лог-файл PHP  
