# hydra

В рамках тестового задания необходимо установить и настроить RADIUS-сервер (FreeRADIUS) в связке с БД MySQL, а также проверить его работу с помощью утилит radtest и radclient.
 
 План действий:
 1) разобраться с ролью RADIUS-взаимодействия в сетях операторов связи;
 2) установить Linux (Debian или Ubuntu);
 3) установить FreeRADIUS (FR);
 4) установить MySQL;
 5) настроить связь FreeRADIUS-MySQL с помощью модуля rlm_sql;
 6) разобраться в структуре конфигурации FR и БД для работы с rlm_sql;
 7) выполнить тестовую авторизацию с помощью radtest; разобраться с механизмом обработки запроса;
 8) настроить выдачу дополнительных атрибутов в результате авторизации (например, Framed-IP-Address);
 9) выполнить отправку аккаунтинг-запросов с помощью radclient: убедиться, что сессии в БД открываются (Start), обновляются (Interim-Update) и завершаются (Stop); разобраться с механизмом обработки запросов.
 
 Задание можно выполнять где удобно: на своей рабочей станции, в облаке и т.п.
 Продемонстрировать результаты нужно лично во время встречи. Никакие отчеты или доступы к результатам до следующей встречи нам не нужны - мы хотим, чтобы Вы сами показали и рассказали как все работает. :)
 Пожалуйста, подтвердите получение задания ответом на это письмо. Если у вас есть вопросы, задайте их в ответе.

## 1) разобраться с ролью RADIUS-взаимодействия в сетях операторов связи;
## 2) установить Linux (Debian или Ubuntu);
## 3) установить FreeRADIUS (FR);

Устанавливаем обновления системы и устанавливаем FreeRADIUS из официального репозитория Ububntu
```
$ sudo apt update
$ sudo apt upgrade
$ sudo apt-get install freeradius freeradius-mysql freeradius-utils
$ sudo systemctl enable freeradius.service
```
Проверям что FreeRADIUS установился и работает
```
$ sudo systemctl status freeradius

● freeradius.service - FreeRADIUS multi-protocol policy server
     Loaded: loaded (/lib/systemd/system/freeradius.service; disabled; vendor preset: enabled)
     Active: active (running) since Sat 2022-05-21 23:38:20 UTC; 26min ago
       Docs: man:radiusd(8)
             man:radiusd.conf(5)
             http://wiki.freeradius.org/
             http://networkradius.com/doc/
   Main PID: 20203 (freeradius)
     Status: "Processing requests"
      Tasks: 6 (limit: 1066)
     Memory: 78.5M
     CGroup: /system.slice/freeradius.service
             └─20203 /usr/sbin/freeradius -f

мая 21 23:38:20 hydra freeradius[20191]: tls: Using cached TLS configuration from previous invocation
мая 21 23:38:20 hydra freeradius[20191]: tls: Using cached TLS configuration from previous invocation
мая 21 23:38:20 hydra freeradius[20191]: rlm_mschap (mschap): using internal authentication
мая 21 23:38:20 hydra freeradius[20191]: rlm_cache (cache_eap): Driver rlm_cache_rbtree (module rlm_cache_rbtree) loaded and linked
мая 21 23:38:20 hydra freeradius[20191]: Ignoring "sql" (see raddb/mods-available/README.rst)
мая 21 23:38:20 hydra freeradius[20191]: Ignoring "ldap" (see raddb/mods-available/README.rst)
мая 21 23:38:20 hydra freeradius[20191]:  # Skipping contents of 'if' as it is always 'false' -- /etc/freeradius/3.0/sites-enabled/inner-tunnel:336
мая 21 23:38:20 hydra freeradius[20191]: radiusd: #### Skipping IP addresses and Ports ####
мая 21 23:38:20 hydra freeradius[20191]: Configuration appears to be OK
мая 21 23:38:20 hydra systemd[1]: Started FreeRADIUS multi-protocol policy server.
```

## 4) установить MySQL
```
$ sudo apt-get mysql-server mysql-client
```
После завешения установки подключаюсь клиентом с верверу MySQL
```
$ sudo mysql -u root -p
```
В консоли БД:
- Создаю базу данных radius`CREATE DATABASE radius CHARACTER SET UTF8 COLLATE UTF8_BIN;`
- Создаю пользователя radius `CREATE USER 'radius'@'%' IDENTIFIED BY 'radiuspass';`
- Даю разрешения на базу данных `GRANT ALL PRIVILEGES ON radius.* TO 'radius'@'%';`
- Применить права `FLUSH PRIVILEGES;`
- Для выхода `QUIT;`

## 5) настроить связь FreeRADIUS-MySQL с помощью модуля rlm_sql
Импортирую схему базы данных
```
$ sudo -i 
# mysql -u root -p radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
```
Подключаю модуль
```
$ sudo ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/
```
Меняю владельца файлов
```
$ sudo chgrp -h freerad /etc/freeradius/3.0/mods-available/sql
$ sudo chown -R freerad:freerad /etc/freeradius/3.0/mods-enabled/sql
```

## 6) разобраться в структуре конфигурации FR и БД для работы с rlm_sql;
В файле `/etc/freeradius/3.0/mods-available/sql` меняю настройки:
```
	dialect = "mysql"
	driver = "rlm_sql_mysql"
        server = "localhost"
        port = 3306
        login = "radius"
        password = "radiuspass"
 
```

## 7) выполнить тестовую авторизацию с помощью radtest; разобраться с механизмом обработки запроса;
## 8) настроить выдачу дополнительных атрибутов в результате авторизации (например, Framed-IP-Address);
Добавляю пользователя в БД
```
insert into radcheck (username,attribute,op,value) values("fred", "Cleartext-Password", ":=", "fredpass");
```
Добавляю аттриьут пользователю
```
insert into radreply (username,attribute,op,value) values("fred", "Framed-IP-Address", ":=", "1.2.3.4");
```
Выполняю тестовую авторизацию
```
radtest fred fredpass 127.0.0.1 10 testing123
```
Где `fred` имя пользователя, `fredpass` его пароль, `127.0.0.1` адрес сервера freeRADIUS, `10` NAS port, `testing123` секрет сервера по умолчанию. Получаю ответ
```
Sent Access-Request Id 226 from 0.0.0.0:36802 to 127.0.0.1:1812 length 74
        User-Name = "fred"
        User-Password = "fredpass"
        NAS-IP-Address = 127.0.1.1
        NAS-Port = 1
        Message-Authenticator = 0x00
        Cleartext-Password = "fredpass"
Received Access-Accept Id 226 from 127.0.0.1:1812 to 127.0.0.1:36802 length 26
        Framed-IP-Address = 1.2.3.4
```

## 9) выполнить отправку аккаунтинг-запросов с помощью radclient: убедиться, что сессии в БД открываются (Start), обновляются (Interim-Update) и завершаются (Stop); разобраться с механизмом обработки запросов.

Создал три файла `start.txt` `update.txt' `stop.txt`
```
$ cat start.txt 
Packet-Type=4
Packet-Dst-Port=1813
Acct-Session-Id = "4D2BB8AC-00000098"
Acct-Status-Type = Start
Acct-Authentic = RADIUS
User-Name = "fred" 
NAS-Port = 0
Called-Station-Id = "00-02-6F-AA-AA-AA:My Wireless"
Calling-Station-Id = "00-1C-B3-AA-AA-AA"
NAS-Port-Type = Wireless-802.11
Connect-Info = "CONNECT 48Mbps 802.11b"

$ cat update.txt 
Packet-Type=4
Packet-Dst-Port=1813
Acct-Session-Id = "4D2BB8AC-00000098"
Acct-Status-Type = Interim-Update
Acct-Authentic = RADIUS
User-Name = "fred"
NAS-Port = 0
Called-Station-Id = "00-02-6F-AA-AA-AA:My Wireless"
Calling-Station-Id = "00-1C-B3-AA-AA-AA"
NAS-Port-Type = Wireless-802.11
Connect-Info = "CONNECT 48Mbps 802.11b"
Acct-Session-Time = 11
Acct-Input-Packets = 15
Acct-Output-Packets = 3
Acct-Input-Octets = 1407
Acct-Output-Octets = 467

cat stop.txt 
Packet-Type=4
Packet-Dst-Port=1813
Acct-Session-Id = "4D2BB8AC-00000098"
Acct-Status-Type = Stop
Acct-Authentic = RADIUS
User-Name = "fred"
NAS-Port = 0
Called-Station-Id = "00-02-6F-AA-AA-AA:My Wireless"
Calling-Station-Id = "00-1C-B3-AA-AA-AA"
NAS-Port-Type = Wireless-802.11
Connect-Info = "CONNECT 48Mbps 802.11b"
Acct-Session-Time = 30
Acct-Input-Packets = 25
Acct-Output-Packets = 7
Acct-Input-Octets = 3407
Acct-Output-Octets = 867
Acct-Terminate-Cause = User-Request
```
Для отправки запроса начала сессии `Start` выполняю команду
```
$ radclient -x 127.0.0.1 acct testing123 -f start.txt 
Sent Accounting-Request Id 242 from 0.0.0.0:40199 to 127.0.0.1:1813 length 143
        Packet-Type = Accounting-Request
        Packet-Dst-Port = 1813
        Acct-Session-Id = "4D2BB8AC-00000098"
        Acct-Status-Type = Start
        Acct-Authentic = RADIUS
        User-Name = "fred"
        NAS-Port = 0
        Called-Station-Id = "00-02-6F-AA-AA-AA:My Wireless"
        Calling-Station-Id = "00-1C-B3-AA-AA-AA"
        NAS-Port-Type = Wireless-802.11
        Connect-Info = "CONNECT 48Mbps 802.11b"
Received Accounting-Response Id 242 from 127.0.0.1:1813 to 127.0.0.1:40199 length 20
```
Данные по сессиям хранятся в таблице `radacct`
```
$ echo "select username, acctstarttime, acctstoptime, acctsessiontime from radacct;"|mysql -u radius -p radius
Enter password: 
username        acctstarttime   acctstoptime    acctsessiontime
fred    2022-05-22 14:34:57     NULL    0
```

Делаю `Interim-Update`
```
$ radclient -x 127.0.0.1 acct testing123 -f update.txt 
Sent Accounting-Request Id 70 from 0.0.0.0:49089 to 127.0.0.1:1813 length 173
        Packet-Type = Accounting-Request
        Packet-Dst-Port = 1813
        Acct-Session-Id = "4D2BB8AC-00000098"
        Acct-Status-Type = Interim-Update
        Acct-Authentic = RADIUS
        User-Name = "fred"
        NAS-Port = 0
        Called-Station-Id = "00-02-6F-AA-AA-AA:My Wireless"
        Calling-Station-Id = "00-1C-B3-AA-AA-AA"
        NAS-Port-Type = Wireless-802.11
        Connect-Info = "CONNECT 48Mbps 802.11b"
        Acct-Session-Time = 11
        Acct-Input-Packets = 15
        Acct-Output-Packets = 3
        Acct-Input-Octets = 1407
        Acct-Output-Octets = 467
Received Accounting-Response Id 70 from 127.0.0.1:1813 to 127.0.0.1:49089 length 20
```
Снова смотрю в БД
```
$ echo "select username, acctstarttime, acctstoptime, acctsessiontime from radacct;
"|mysql -u radius -p radius
Enter password: 
username        acctstarttime   acctstoptime    acctsessiontime
fred    2022-05-22 14:34:57     NULL    11
```
Делаю `Stop`
```
radclient -x 127.0.0.1 acct testing123 -f stop.txt 
Sent Accounting-Request Id 137 from 0.0.0.0:56058 to 127.0.0.1:1813 length 179
        Packet-Type = Accounting-Request
        Packet-Dst-Port = 1813
        Acct-Session-Id = "4D2BB8AC-00000098"
        Acct-Status-Type = Stop
        Acct-Authentic = RADIUS
        User-Name = "fred"
        NAS-Port = 0
        Called-Station-Id = "00-02-6F-AA-AA-AA:My Wireless"
        Calling-Station-Id = "00-1C-B3-AA-AA-AA"
        NAS-Port-Type = Wireless-802.11
        Connect-Info = "CONNECT 48Mbps 802.11b"
        Acct-Session-Time = 30
        Acct-Input-Packets = 25
        Acct-Output-Packets = 7
        Acct-Input-Octets = 3407
        Acct-Output-Octets = 867
        Acct-Terminate-Cause = User-Request
Received Accounting-Response Id 137 from 127.0.0.1:1813 to 127.0.0.1:56058 length 20
```

Прверяю БД
```
echo "select username, acctstarttime, acctstoptime, acctsessiontime from radacct;
"|mysql -u radius -p radius
Enter password: 
username        acctstarttime   acctstoptime    acctsessiontime
fred    2022-05-22 14:34:57     2022-05-22 14:40:04     30
```
