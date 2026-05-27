# Домашнее задание "Systemd — создание unit-файла"

## Задание

1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).
2. Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).
3. Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.
---

## Выполнение задания

### 1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова


Cоздаём файл с конфигурацией для сервиса в директории /etc/default - из неё сервис будет брать необходимые переменные.

```bash
root@client:/etc/default# touch watchlog
root@client:/etc/default# vi watchlog

Содержимое:
# Configuration file for my watchlog service
# Place it to /etc/default

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```

Создаем /var/log/watchlog.log и пишем туда строки на своё усмотрение, плюс ключевое слово ‘ALERT’
```bash
root@client:/etc/default# touch /var/log/watchlog.log
root@client:/etc/default# vi  /var/log/watchlog.log
```

Создадим скрипт:
```bash
root@client:/etc/default# cat > /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi
```

Команда logger отправляет лог в системный журнал.
Добавим права на запуск файла:
```bash
root@client:/etc/default# chmod +x /opt/watchlog.sh
```

Создадим юнит для сервиса:
```bash
root@client:/etc/default# cat > /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```

Создадим юнит для таймера:
```bash
root@client:/etc/default# cat > /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
```

Затем запускаем все что сделали:
```bash
root@client:/etc/default# systemctl start watchlog.service
root@client:/etc/default# systemctl start watchlog.timer

root@client:/etc/default# systemctl status watchlog.timer
● watchlog.timer - Run watchlog script every 30 second
     Loaded: loaded (/etc/systemd/system/watchlog.timer; disabled; preset: enabled)
     Active: active (waiting) since Wed 2026-05-27 18:58:35 UTC; 5min ago
    Trigger: Wed 2026-05-27 19:04:19 UTC; 5s left
   Triggers: ● watchlog.service
```

И убедиться в результате:
```bash
root@client:/etc/default# tail -n 1000 /var/log/syslog  | grep word
2026-05-27T19:02:48.282344+00:00 client root: Wed May 27 07:02:48 PM UTC 2026: I found word, Master!
2026-05-27T19:03:49.242550+00:00 client root: Wed May 27 07:03:49 PM UTC 2026: I found word, Master!
2026-05-27T19:04:44.581873+00:00 client root: Wed May 27 07:04:44 PM UTC 2026: I found word, Master!
2026-05-27T19:05:44.638671+00:00 client root: Wed May 27 07:05:44 PM UTC 2026: I found word, Master!
2026-05-27T19:06:34.624665+00:00 client root: Wed May 27 07:06:34 PM UTC 2026: I found word, Master!
```

### 2. Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта

Устанавливаем spawn-fcgi и необходимые для него пакеты:
```bash
root@client:/etc/default#  apt install spawn-fcgi php php-cgi php-cli \
 apache2 libapache2-mod-fcgid -y
```
Сам Init скрипт, который будем переписывать, можно найти здесь: https://gist.github.com/cea2k/1318020 

Но перед этим необходимо создать файл с настройками для будущего сервиса в файле /etc/spawn-fcgi/fcgi.conf.
```bash
root@client:/etc/default# mkdir /etc/spawn-fcgi
root@client:/etc/default# touch /etc/spawn-fcgi/fcgi.conf
root@client:/etc/default# cat > /etc/spawn-fcgi/fcgi.conf
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
```

Создаем юнит-файл:
```bash
root@client:/etc/default# cat > /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```

Убеждаемся, что все успешно работает:
```bash
root@client:/etc/default# systemctl start spawn-fcgi
root@client:/etc/default# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset: enabled)
     Active: active (running) since Wed 2026-05-27 19:14:31 UTC; 4s ago
   Main PID: 11372 (php-cgi)
      Tasks: 33 (limit: 1056)
     Memory: 19.9M (peak: 20.1M)
        CPU: 104ms
     CGroup: /system.slice/spawn-fcgi.service
```
### 3. Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно

Установим Nginx из стандартного репозитория:
```bash
root@client:/etc/default# apt update
root@client:/etc/default# apt install nginx -y
```

Для запуска нескольких экземпляров сервиса модифицируем исходный service для использования различной конфигурации, а также PID-файлов. Для этого создадим новый Unit для работы с шаблонами (/etc/systemd/system/nginx@.service):
```bash
root@client:/etc/default# cat > /etc/systemd/system/nginx@.service
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
```

Далее необходимо создать два файла конфигурации (/etc/nginx/nginx-first.conf, /etc/nginx/nginx-second.conf). Их можно сформировать из стандартного конфига /etc/nginx/nginx.conf, с модификацией путей до PID-файлов и разделением по портам:

```bash
Первый инстанс - порт 9001

cat > /etc/nginx/nginx-first.conf << 'EOF'
user www-data;
worker_processes auto;
pid /run/nginx-first.pid;
error_log /var/log/nginx/error-first.log;

events {
    worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    access_log /var/log/nginx/access-first.log;
    
    server {
        listen 9001;
        server_name localhost;
        
        location / {
            root /var/www/html-first;
            index index.html index.htm;
        }
    }
}
EOF

Второй инстанс - порт 9002

cat > /etc/nginx/nginx-second.conf << 'EOF'
user www-data;
worker_processes auto;
pid /run/nginx-second.pid;
error_log /var/log/nginx/error-second.log;

events {
    worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    access_log /var/log/nginx/access-second.log;
    
    server {
        listen 9002;
        server_name localhost;
        
        location / {
            root /var/www/html-second;
            index index.html index.htm;
        }
    }
}
EOF
```

Этого достаточно для успешного запуска сервисов.
Проверим работу:
```bash

root@client:/etc/default# systemctl start nginx@first
root@client:/etc/default# systemctl start nginx@second
root@client:/etc/default# systemctl status nginx@second
● nginx@second.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; preset: enabled)
     Active: active (running) since Wed 2026-05-27 19:29:10 UTC; 21s ago
       Docs: man:nginx(8)
    Process: 12114 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-second.conf -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 12116 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 12117 (nginx)
      Tasks: 2 (limit: 1056)
     Memory: 1.7M (peak: 1.9M)
        CPU: 35ms
     CGroup: /system.slice/system-nginx.slice/nginx@second.service
             ├─12117 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;"
             └─12118 "nginx: worker process"

May 27 19:29:10 client systemd[1]: Starting nginx@second.service - A high performance web server and a reverse proxy server...
May 27 19:29:10 client systemd[1]: Started nginx@second.service - A high performance web server and a reverse proxy server.
root@client:/etc/default# systemctl status nginx@first
● nginx@first.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; preset: enabled)
     Active: active (running) since Wed 2026-05-27 19:29:03 UTC; 40s ago
       Docs: man:nginx(8)
    Process: 12106 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-first.conf -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 12108 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 12109 (nginx)
      Tasks: 2 (limit: 1056)
     Memory: 1.7M (peak: 1.9M)
        CPU: 26ms
     CGroup: /system.slice/system-nginx.slice/nginx@first.service
             ├─12109 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;"
             └─12110 "nginx: worker process"

May 27 19:29:03 client systemd[1]: Starting nginx@first.service - A high performance web server and a reverse proxy server...
May 27 19:29:03 client systemd[1]: Started nginx@first.service - A high performance web server and a reverse proxy server.
```
Проверить можно несколькими способами, например, посмотреть, какие порты слушаются:
```bash
root@client:/etc/default# ss -tnulp | grep nginx
tcp   LISTEN 0      511                 0.0.0.0:9002      0.0.0.0:*    users:(("nginx",pid=12118,fd=5),("nginx",pid=12117,fd=5))                                                                                                                                           
tcp   LISTEN 0      511                 0.0.0.0:9001      0.0.0.0:*    users:(("nginx",pid=12110,fd=5),("nginx",pid=12109,fd=5))    
```
Или просмотреть список процессов:
```bash
root@client:/etc/default# ps afx | grep nginx
  12148 pts/1    S+     0:00                          \_ grep --color=auto nginx
  12109 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;
  12110 ?        S      0:00  \_ nginx: worker process
  12117 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;
  12118 ?        S      0:00  \_ nginx: worker process
```
