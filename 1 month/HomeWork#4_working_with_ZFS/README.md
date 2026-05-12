# Домашнее задание "Работа с ZFS"

## Задание

1. Определить алгоритм с наилучшим сжатием:
  - Определить какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);
  - создать 4 файловых системы на каждой применить свой алгоритм сжатия;
  - для сжатия использовать либо текстовый файл, либо группу файлов.
2. Определить настройки пула.
С помощью команды zfs import собрать pool ZFS.
Командами zfs определить настройки:
    - размер хранилища;
    - тип pool;
    - значение recordsize;
    - какое сжатие используется;
    - какая контрольная сумма используется.
3. Работа со снапшотами:
  - скопировать файл из удаленной директории;
  - восстановить файл локально. zfs receive;
  - найти зашифрованное сообщение в файле secret_message.

---

## Выполнение

Добавляем 8 дополнительных дисков по 512 MB для проведения работ:
<details>

```bash
root@server:/home/user# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   10G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0  8.2G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0  8.2G  0 lvm  /
sdb                         8:16   0  512M  0 disk 
sdc                         8:32   0  512M  0 disk 
sdd                         8:48   0  512M  0 disk 
sde                         8:64   0  512M  0 disk 
sdf                         8:80   0  512M  0 disk 
sdg                         8:96   0  512M  0 disk 
sdh                         8:112  0  512M  0 disk 
sdi                         8:128  0  512M  0 disk 

```
</details>

### 1. Определение алгоритма с наилучшим сжатием.


Установим пакет утилит для ZFS:
```bash
root@server:/home/user# sudo apt install zfsutils-linux
```
Создаём пул из двух дисков в режиме RAID 1:
```bash
root@server:/home/user# zpool create otus1 mirror /dev/sdb /dev/sdc
```

Создадим ещё 3 пула: 
```bash
root@server:/home/user# zpool create otus2 mirror /dev/sdd /dev/sde
root@server:/home/user# zpool create otus3 mirror /dev/sdf /dev/sdg
root@server:/home/user# zpool create otus4 mirror /dev/sdh /dev/sdi
```

Смотрим информацию о пулах: zpool list
```bash
root@server:/home/user# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M   104K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M   105K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -

root@server:/home/user# zpool status
  pool: otus1
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus1       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus2       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus3       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus4       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdi     ONLINE       0     0     0

errors: No known data errors
```

Команда zpool status показывает информацию о каждом диске, состоянии сканирования и об ошибках чтения, записи и совпадения хэш-сумм. Команда zpool list показывает информацию о размере пула, количеству занятого и свободного места, дедупликации и т.д. 

Добавим разные алгоритмы сжатия в каждую файловую систему:
Алгоритм lzjb: zfs set compression=lzjb otus1
Алгоритм lz4:  zfs set compression=lz4 otus2
Алгоритм gzip: zfs set compression=gzip-9 otus3
Алгоритм zle:  zfs set compression=zle otus4
```bash
root@server:/home/user# zfs set compression=lzjb otus1
root@server:/home/user# zfs set compression=lz4 otus2
root@server:/home/user# zfs set compression=gzip-9 otus3
root@server:/home/user# zfs set compression=zle otus4
```

Проверим, что все файловые системы имеют разные методы сжатия:
```bash
root@server:/home/user# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```

Сжатие файлов будет работать только с файлами, которые были добавлены после включение настройки сжатия. 
Скачаем один и тот же текстовый файл во все пулы: 
```bash
root@server:/var/log# for i in {1..4}; do cp -r dpkg.log /otus$i; done
```

Проверим, что файл был скачан во все пулы:
```bash
root@server:/var/log# ls -l /otus*
/otus1:
total 123
-rw-r--r-- 1 root root 719180 May 12 06:05 dpkg.log

/otus2:
total 115
-rw-r--r-- 1 root root 719180 May 12 06:05 dpkg.log

/otus3:
total 64
-rw-r--r-- 1 root root 719180 May 12 06:05 dpkg.log

/otus4:
total 707
-rw-r--r-- 1 root root 719180 May 12 06:05 dpkg.log
```

Самый оптимальный метод сжатия у нас используется в пуле otus3.
Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:
```bash
root@server:/var/log# zfs list
NAME    USED  AVAIL  REFER  MOUNTPOINT
otus1   296K   352M   146K  /otus1
otus2   254K   352M   138K  /otus2
otus3   208K   352M    87K  /otus3
otus4   852K   351M   730K  /otus4


root@server:/var/log# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         4.52x                  -
otus2  compressratio         5.01x                  -
otus3  compressratio         7.17x                  -
otus4  compressratio         1.08x                  -
```
Таким образом, у нас получается, что алгоритм gzip-9 самый эффективный по сжатию. 


### 2. Определение настроек пула.
Скачиваем архив в домашний каталог: 
```bash
root@server:~# wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download' 
```
Разархивируем его:
```bash
root@server:~# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```

Проверим, возможно ли импортировать данный каталог в пул:
```bash
root@server:~# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
        (Note that they may be intentionally disabled if the
        'compatibility' property is set.)
 action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
 config:

        otus                         ONLINE
          mirror-0                   ONLINE
            /root/zpoolexport/filea  ONLINE
            /root/zpoolexport/fileb  ONLINE
```

Данный вывод показывает нам имя пула, тип raid и его состав. 
Сделаем импорт данного пула к нам в ОС:
```bash
root@server:~# zpool import -d zpoolexport/ otus
root@server:~# zpool status otus
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
        The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(7) for details.
config:

        NAME                         STATE     READ WRITE CKSUM
        otus                         ONLINE       0     0     0
          mirror-0                   ONLINE       0     0     0
            /root/zpoolexport/filea  ONLINE       0     0     0
            /root/zpoolexport/fileb  ONLINE       0     0     0
```

Команда zpool status выдаст нам информацию о составе импортированного пула.
Если у Вас уже есть пул с именем otus, то можно поменять его имя во время импорта: zpool import -d zpoolexport/ otus newotus
Далее нам нужно определить настройки: zpool get all otus
Запрос сразу всех параметром файловой системы: zfs get all otus
```bash
root@server:~# zfs get all otus
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.04M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclmode               discard                default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.01M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1020K                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              on                     default
otus  redundant_metadata    all                    default
otus  overlay               on                     default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default
```

Можно уточнить конкретный параметр, например:
Размер: zfs get available otus
```bash
root@server:~# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
```

Тип: zfs get readonly otus
```bash
root@server:~# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
```

По типу FS мы можем понять, что позволяет выполнять чтение и запись
Значение recordsize: zfs get recordsize otus
```bash
root@server:~# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
```

Тип сжатия (или параметр отключения): zfs get compression otus
```bash
root@server:~# zfs get compression otus
NAME  PROPERTY     VALUE           SOURCE
otus  compression  zle             local
```

Тип контрольной суммы: zfs get checksum otus
```bash
root@server:~# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```
### 3. Работа со снапшотом.
Скачаем файл:
```bash
root@server:~# wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download
```

Восстановим файловую систему из снапшота:
```bash
zfs receive otus/test@today < otus_task2.file
```
Далее, ищем в каталоге /otus/test файл с именем “secret_message”:
```bash
root@server:~# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
```

Смотрим содержимое найденного файла:
```bash
root@server:~# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/
```

Тут мы видим ссылку на курс OTUS, задание выполнено.

