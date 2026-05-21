# Домашнее задание "Работа с RPM"

## Задание

- Создать свой RPM пакет (можно взять свое приложение, либо собрать, например,
Apache с определенными опциями).
- Создать свой репозиторий и разместить там ранее собранный RPM.
---

## Выполнение домашнего задания

Работа выполняется на ОС Rocky Linux 9.7 (Blue Onyx)

### 1. Создать свой RPM пакет.

-Устанавливаем необходимые для работы пакеты:
```bash
[root@rocky user]# yum install -y wget rpmdevtools rpm-build createrepo yum-utils cmake gcc git nano
```

Для примера будет использован пакет Nginx и соберем его с дополнительным модулем ngx_broli

Загрузим SRPM пакет Nginx для дальнейшей работы над ним:
```bash
[root@rocky user]# mkdir rpm && cd rpm
[root@rocky rpm]# yumdownloader --source nginx
enabling baseos-source repository
enabling appstream-source repository
enabling extras-source repository
Rocky Linux 9 - BaseOS - Source                                                                                                                   111 kB/s | 423 kB     00:03    
Rocky Linux 9 - AppStream - Source                                                                                                                 39 kB/s | 945 kB     00:24    
Rocky Linux 9 - Extras Source                                                                                                                     9.2 kB/s |  14 kB     00:01    
[MIRROR] nginx-1.20.1-24.el9_7.3.rocky.0.1.src.rpm: Status code: 404 for https://ru.mirrors.cicku.me/rocky/9.7/AppStream/source/tree/Packages/n/nginx-1.20.1-24.el9_7.3.rocky.0.1.src.rpm (IP: 172.65.90.45)
(1/2): nginx-1.20.1-24.el9_7.2.rocky.0.1.src.rpm                                                                                                  374 kB/s | 1.1 MB     00:02    
(2/2): nginx-1.20.1-24.el9_7.3.rocky.0.1.src.rpm                                                                                                   32 kB/s | 1.1 MB     00:33    
[root@rocky rpm]# ll
total 3080
-rw-r--r--. 1 root root 1119270 May 21 01:43 nginx-1.20.1-24.el9_7.2.rocky.0.1.src.rpm
-rw-r--r--. 1 root root 1120276 May 21 01:43 nginx-1.20.1-24.el9_7.3.rocky.0.1.src.rpm
```

При установке такого пакета в домашней директории создается дерево каталогов для сборки, далее поставим все зависимости для сборки пакета Nginx:
```bash
[root@rocky rpm]# rpm -Uvh nginx*.src.rpm
[root@rocky rpm]# sudo yum-builddep -y nginx
```

Далее скачаем исходный код модуля ngx_brotli — он потребуется при сборке:
```bash

# Скачиваем основной репозиторий через wget
wget https://github.com/google/ngx_brotli/archive/refs/heads/master.zip -O ngx_brotli.zip

# Распаковываем
unzip ngx_brotli.zip
mv ngx_brotli-master ngx_brotli

# Скачиваем brotli (репозиторий Google, который нужен для модуля)
wget https://github.com/google/brotli/archive/refs/heads/master.zip -O brotli.zip

# Создаём структуру и распаковываем
mkdir -p ngx_brotli/deps
unzip brotli.zip
mv brotli-master ngx_brotli/deps/brotli

# Чистим архивы
rm -f ngx_brotli.zip brotli.zip

[root@rocky ~]# cd ngx_brotli/deps/brotli
[root@rocky brotli]# mkdir out && cd out
``` 

● Собираем модуль ngx_brotli:

```bash
[root@rocky build]# cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..
-- Build type is 'Release'
-- Compiler is not EMSCRIPTEN
Test file tests/testdata/alice29.txt does not exist; OK on tarball builds; consider running scripts/download_testdata.sh before configuring.
Test file tests/testdata/asyoulik.txt does not exist; OK on tarball builds; consider running scripts/download_testdata.sh before configuring.
Test file tests/testdata/lcet10.txt does not exist; OK on tarball builds; consider running scripts/download_testdata.sh before configuring.
Test file tests/testdata/plrabn12.txt does not exist; OK on tarball builds; consider running scripts/download_testdata.sh before configuring.
-- Configuring done (0.0s)
-- Generating done (0.0s)
CMake Warning:
  Manually-specified variables were not used by the project:

    CMAKE_CXX_FLAGS




[root@rocky build]# cmake --build . --config Release -j 2 --target brotlienc
[  3%] Building C object CMakeFiles/brotlicommon.dir/c/common/constants.c.o
[  6%] Building C object CMakeFiles/brotlicommon.dir/c/common/context.c.o
[  9%] Building C object CMakeFiles/brotlicommon.dir/c/common/dictionary.c.o
[ 12%] Building C object CMakeFiles/brotlicommon.dir/c/common/platform.c.o
[ 16%] Building C object CMakeFiles/brotlicommon.dir/c/common/shared_dictionary.c.o
[ 19%] Building C object CMakeFiles/brotlicommon.dir/c/common/transform.c.o
[ 22%] Linking C static library libbrotlicommon.a
[ 22%] Built target brotlicommon
[ 25%] Building C object CMakeFiles/brotlienc.dir/c/enc/backward_references.c.o
[ 29%] Building C object CMakeFiles/brotlienc.dir/c/enc/backward_references_hq.c.o
[ 32%] Building C object CMakeFiles/brotlienc.dir/c/enc/bit_cost.c.o
[ 35%] Building C object CMakeFiles/brotlienc.dir/c/enc/block_splitter.c.o
[ 38%] Building C object CMakeFiles/brotlienc.dir/c/enc/brotli_bit_stream.c.o
[ 41%] Building C object CMakeFiles/brotlienc.dir/c/enc/cluster.c.o
[ 45%] Building C object CMakeFiles/brotlienc.dir/c/enc/command.c.o
[ 48%] Building C object CMakeFiles/brotlienc.dir/c/enc/compound_dictionary.c.o
[ 51%] Building C object CMakeFiles/brotlienc.dir/c/enc/compress_fragment.c.o
[ 54%] Building C object CMakeFiles/brotlienc.dir/c/enc/compress_fragment_two_pass.c.o
[ 58%] Building C object CMakeFiles/brotlienc.dir/c/enc/dictionary_hash.c.o
[ 61%] Building C object CMakeFiles/brotlienc.dir/c/enc/encode.c.o
[ 64%] Building C object CMakeFiles/brotlienc.dir/c/enc/encoder_dict.c.o
[ 67%] Building C object CMakeFiles/brotlienc.dir/c/enc/entropy_encode.c.o
[ 70%] Building C object CMakeFiles/brotlienc.dir/c/enc/fast_log.c.o
[ 74%] Building C object CMakeFiles/brotlienc.dir/c/enc/histogram.c.o
[ 77%] Building C object CMakeFiles/brotlienc.dir/c/enc/literal_cost.c.o
[ 80%] Building C object CMakeFiles/brotlienc.dir/c/enc/memory.c.o
[ 83%] Building C object CMakeFiles/brotlienc.dir/c/enc/metablock.c.o
[ 87%] Building C object CMakeFiles/brotlienc.dir/c/enc/static_dict.c.o
[ 90%] Building C object CMakeFiles/brotlienc.dir/c/enc/static_dict_lut.c.o
[ 93%] Building C object CMakeFiles/brotlienc.dir/c/enc/static_init.c.o
[ 96%] Building C object CMakeFiles/brotlienc.dir/c/enc/utf8_util.c.o
[100%] Linking C static library libbrotlienc.a
[100%] Built target brotlienc



[root@rocky build]# cd ../../../..
[root@rocky ~]# 

```

● Нужно поправить сам spec файл, чтобы Nginx собирался с необходимыми нам опциями: находим секцию с параметрами configure (до условий if) и добавляем указание на модуль (не забудьте указать завершающий обратный слэш):
--add-module=/root/ngx_brotli \
```bash
[root@rocky SPECS]# sed -i '/--with-debug \\/a\    --add-module=/root/ngx_brotli \\' nginx.spec
```

Приступаем к сборке RPM пакета:
```bash
[root@rocky SPECS]# cd ~/rpmbuild/SPECS/
[root@rocky SPECS]# rpmbuild -ba nginx.spec -D 'debug_package %{nil}'
...
Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.kOTpGm
+ umask 022
+ cd /root/rpmbuild/BUILD
+ cd nginx-1.20.1
+ /usr/bin/rm -rf /root/rpmbuild/BUILDROOT/nginx-1.20.1-24.el9.3.rocky.0.1.x86_64
+ RPM_EC=0
++ jobs -p
+ exit 0
```

Убедимся, что пакеты создались:
```bash
[root@rocky ~]# ll rpmbuild/RPMS/x86_64/
-rw-r--r--. 1 root root  37668 May 21 05:44 nginx-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm
-rw-r--r--. 1 root root 595372 May 21 05:44 nginx-core-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm
-rw-r--r--. 1 root root 762086 May 21 05:44 nginx-mod-devel-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm
-rw-r--r--. 1 root root  20733 May 21 05:44 nginx-mod-http-image-filter-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm
-rw-r--r--. 1 root root  32247 May 21 05:44 nginx-mod-http-perl-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm
-rw-r--r--. 1 root root  19533 May 21 05:44 nginx-mod-http-xslt-filter-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm
-rw-r--r--. 1 root root  55134 May 21 05:44 nginx-mod-mail-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm
-rw-r--r--. 1 root root  81729 May 21 05:44 nginx-mod-stream-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm
```

Копируем пакеты в общий каталог:
```bash
[root@rocky ~]# cp ~/rpmbuild/RPMS/noarch/* ~/rpmbuild/RPMS/x86_64/
[root@rocky ~]# cd ~/rpmbuild/RPMS/x86_64
```
Устанавливаем наш пакет и убеждаемся, что nginx работает:
```bash
[root@rocky x86_64]# # yum localinstall -y *.rpm
[root@rocky x86_64]# systemctl start nginx
[root@rocky x86_64]# systemctl status nginx

[root@rocky x86_64]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Thu 2026-05-21 05:51:45 EDT; 11s ago
    Process: 45459 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 45460 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 45461 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 45462 (nginx)
      Tasks: 2 (limit: 4596)
     Memory: 6.0M (peak: 6.3M)
        CPU: 41ms
     CGroup: /system.slice/nginx.service
             ├─45462 "nginx: master process /usr/sbin/nginx"
             └─45463 "nginx: worker process"

May 21 05:51:45 rocky systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 21 05:51:45 rocky nginx[45460]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 21 05:51:45 rocky nginx[45460]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 21 05:51:45 rocky systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Далее будем использовать его для доступа к своему репозиторию.

### 2. Создать свой репозиторий и разместить там ранее собранный RPM

Приступим к созданию своего репозитория. Директория для статики у Nginx по умолчанию /usr/share/nginx/html. Создадим там каталог repo:
```bash
[root@rocky x86_64]# mkdir /usr/share/nginx/html/repo
```
Скопируем туда наши собранные RPM-пакеты:
```bash
[root@rocky x86_64]# cp ~/rpmbuild/RPMS/x86_64/*.rpm /usr/share/nginx/html/repo/
```
● Инициализируем репозиторий:
```bash
[root@rocky x86_64]# createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 10 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```
Для прозрачности настроим в NGINX доступ к листингу каталога. В файле /etc/nginx/nginx.conf в блоке server добавим следующие директивы:

	index index.html index.htm;
	autoindex on;

Проверяем синтаксис и перезапускаем NGINX:
```bash
[root@rocky x86_64]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@rocky x86_64]# nginx -s reload
```

Проверяем с помощью curl:
```bash
[root@rocky x86_64]# curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          21-May-2026 10:06                   -
<a href="nginx-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm">nginx-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm</a>         21-May-2026 10:05               37668
<a href="nginx-all-modules-1.20.1-24.el9.3.rocky.0.1.noarch.rpm">nginx-all-modules-1.20.1-24.el9.3.rocky.0.1.noa..&gt;</a> 21-May-2026 10:05                8753
<a href="nginx-core-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm">nginx-core-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm</a>    21-May-2026 10:05              595372
<a href="nginx-filesystem-1.20.1-24.el9.3.rocky.0.1.noarch.rpm">nginx-filesystem-1.20.1-24.el9.3.rocky.0.1.noar..&gt;</a> 21-May-2026 10:05               10340
<a href="nginx-mod-devel-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm">nginx-mod-devel-1.20.1-24.el9.3.rocky.0.1.x86_6..&gt;</a> 21-May-2026 10:05              762086
<a href="nginx-mod-http-image-filter-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm">nginx-mod-http-image-filter-1.20.1-24.el9.3.roc..&gt;</a> 21-May-2026 10:05               20733
<a href="nginx-mod-http-perl-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm">nginx-mod-http-perl-1.20.1-24.el9.3.rocky.0.1.x..&gt;</a> 21-May-2026 10:05               32247
<a href="nginx-mod-http-xslt-filter-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm">nginx-mod-http-xslt-filter-1.20.1-24.el9.3.rock..&gt;</a> 21-May-2026 10:05               19533
<a href="nginx-mod-mail-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm">nginx-mod-mail-1.20.1-24.el9.3.rocky.0.1.x86_64..&gt;</a> 21-May-2026 10:05               55134
<a href="nginx-mod-stream-1.20.1-24.el9.3.rocky.0.1.x86_64.rpm">nginx-mod-stream-1.20.1-24.el9.3.rocky.0.1.x86_..&gt;</a> 21-May-2026 10:05               81729
</pre><hr></body>
</html>
```
Все готово для того, чтобы протестировать репозиторий.
Добавим его в /etc/yum.repos.d:
```bash
[root@rocky x86_64]# cat >> /etc/yum.repos.d/otus.repo << EOF
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
EOF
```

Убедимся, что репозиторий подключился и посмотрим, что в нем есть:
```bash
[root@rocky x86_64]# yum repolist enabled | grep otus
otus                            otus-linux
```

Добавим пакет в наш репозиторий:
```bash
[root@rocky x86_64]# cd /usr/share/nginx/html/repo/
[root@rocky repo]# wget https://repo.percona.com/yum/percona-release-latest.noarch.rpm
```

Обновим список пакетов в репозитории:
```bash
[root@rocky repo]# createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 11 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
[root@rocky repo]# yum makecache
otus-linux                                                                                                                                        1.3 MB/s | 7.2 kB     00:00    
Rocky Linux 9 - BaseOS                                                                                                                            1.8 kB/s | 4.3 kB     00:02    
Rocky Linux 9 - BaseOS                                                                                                                            670 kB/s |  23 MB     00:35    
Rocky Linux 9 - AppStream                                            112% [=============================================================================-] 8.3 kB/s | 4.8 kB     -Rocky Linux 9 - AppStream                                                                                                                         2.9 kB/s | 4.8 kB     00:01    
Rocky Linux 9 - Extras                                                                                                                            7.2 kB/s | 3.1 kB     00:00    
Metadata cache created.
[root@rocky repo]# yum list | grep otus
percona-release.noarch                               1.0-33                              otus         
[root@rocky repo]# 
```

Так как Nginx у нас уже стоит, установим репозиторий percona-release:
```bash
[root@rocky repo]# yum install -y percona-release.noarch
Installed:
  percona-release-1.0-33.noarch                                                                                                                                                   

Complete!
```
Все прошло успешно. В случае, если потребуется обновить репозиторий (а это
делается при каждом добавлении файлов) снова, то требуется выполнить команду
createrepo /usr/share/nginx/html/repo/.
