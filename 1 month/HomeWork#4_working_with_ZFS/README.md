# Домашнее задание "Работа с LVM"

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

Добавляем 4 диска для проведения работ:
<details>

```bash
root@user:/home/user# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   10G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0  8.2G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0  8.2G  0 lvm  /
sdb                         8:16   0   10G  0 disk 
sdc                         8:32   0    2G  0 disk 
sdd                         8:48   0    1G  0 disk 
sde                         8:64   0    1G  0 disk  

root@user:/home/user# lvmdiskscan 
  /dev/sda2 [       1.75 GiB] 
  /dev/sda3 [      <8.25 GiB] LVM physical volume
  /dev/sdb  [      10.00 GiB] 
  /dev/sdc  [       2.00 GiB] 
  /dev/sdd  [       1.00 GiB] 
  /dev/sde  [       1.00 GiB] 
  4 disks
  1 partition
  0 LVM physical volume whole disks
  1 LVM physical volume
```
</details>

### 1. Уменьшить том под / до 8G.

