# Домашнее задание "Работа с LVM"

## Задание

1. Уменьшить том под / до 8G.
2. Выделить том под /var - сделать в mirror.
3. Выделить том под /home.
4. Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор).
5. /home - сделать том для снапшотов.
 Работа со снапшотами:
  - сгенерить файлы в /home/;
  - снять снапшот;
  - удалить часть файлов;
  - восстановиться со снапшота.
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

Подготовим временный том для / раздела:

root@user:/home/user# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.

root@user:/home/user# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created

root@user:/home/user# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.

Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:

root@user:/home/user# mkfs.ext4 /dev/vg_root/lv_root
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2620416 4k blocks and 655360 inodes
Filesystem UUID: 6105b399-dc7f-48f2-bd49-09e3d5731b08
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

root@user:/home/user# mount /dev/vg_root/lv_root /mnt

Копируем все данные с / раздела в /mnt:

root@user:/home/user# rsync -avxHAX --progress / /mnt/
…

sent 4,778,761,976 bytes  received 2,287,689 bytes  24,836,621.64 bytes/sec
total size is 4,773,506,378  speedup is 1.00

Проверzем что данные скопировались: 
root@user:/home/user# ls /mnt
bin                boot   dev  home  lib64              lost+found  mnt  proc  run   sbin.usr-is-merged  srv       sys  usr
bin.usr-is-merged  cdrom  etc  lib   lib.usr-is-merged  media       opt  root  sbin  snap                swap.img  tmp  var


Затем сконфигурируем grub для того, чтобы при старте перейти в новый /.
Сымитируем текущий root, сделаем в него chroot и обновим grub:

root@user:/home/user# for i in /proc/ /sys/ /dev/ /run/ /boot/; \
 do mount --bind $i /mnt/$i; done

root@user:/home/user# chroot /mnt/

root@user:/# grub-mkconfig -o /boot/grub/grub.cfg
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-7.0.0-070000rc6-generic
Found initrd image: /boot/initrd.img-7.0.0-070000rc6-generic
Found linux image: /boot/vmlinuz-6.8.0-107-generic
Found initrd image: /boot/initrd.img-6.8.0-107-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done


Обновим образ initrd. 
root@user:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-7.0.0-070000rc6-generic


Презагружаемся (предварительно выйдя из chroot), чтобы работать с новым разделом.
root@user:/# reboot -h now

Проверяем что все работы выполнены корректно после перезагрузки:

root@user:/home/user# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   10G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0  8.2G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:1    0  8.2G  0 lvm  
sdb                         8:16   0   10G  0 disk 
└─vg_root-lv_root         252:0    0   10G  0 lvm  /
sdc                         8:32   0    2G  0 disk 
sdd                         8:48   0    1G  0 disk 
sde                         8:64   0    1G  0 disk 
sr0                        11:0    1 1024M  0 rom  


Теперь нам нужно изменить размер старой VG и вернуть на него рут. Для этого удаляем старый LV и создаём новый на 8G:

root@user:/home/user# lvremove /dev/ubuntu-vg/ubuntu-lv
Do you really want to remove and DISCARD active logical volume ubuntu-vg/ubuntu-lv? [y/n]: y
  Logical volume "ubuntu-lv" successfully removed.

root@user:/home/user# lvcreate -n ubuntu-vg/ubuntu-lv -L 8G /dev/ubuntu-vg
WARNING: ext4 signature detected on /dev/ubuntu-vg/ubuntu-lv at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/ubuntu-vg/ubuntu-lv.
  Logical volume "ubuntu-lv" created.

Проделываем на нем те же операции, что и в первый раз:

root@user:/home/user# mkfs.ext4 /dev/ubuntu-vg/ubuntu-lv
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2097152 4k blocks and 524288 inodes
Filesystem UUID: b7aacea5-d3ad-4386-9620-b30c4b54d209
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 


root@user:/home/user# mount /dev/ubuntu-vg/ubuntu-lv /mnt

root@user:/home/user# rsync -avxHAX --progress / /mnt/
sent 4,795,736,673 bytes  received 2,287,870 bytes  21,858,881.74 bytes/sec
total size is 4,790,480,055  speedup is 1.00

Так же как в первый раз cконфигурируем grub.

root@user:/home/user# for i in /proc/ /sys/ /dev/ /run/ /boot/; \
 do mount --bind $i /mnt/$i; done
root@user:/home/user# chroot /mnt/
root@user:/# grub-mkconfig -o /boot/grub/grub.cfg
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-7.0.0-070000rc6-generic
Found initrd image: /boot/initrd.img-7.0.0-070000rc6-generic
Found linux image: /boot/vmlinuz-6.8.0-107-generic
Found initrd image: /boot/initrd.img-6.8.0-107-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done

root@user:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-7.0.0-070000rc6-generic
W: Couldn't identify type of root file system for fsck hook


### 2. Выделить том под /var - сделать в mirror.
Пока не перезагружаемся и не выходим из под chroot - мы можем заодно перенести /var.
Выделить том под /var в зеркало
На свободных дисках создаем зеркало:

root@user:/# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.

root@user:/# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created

root@user:/# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.

Создаем на нем ФС и перемещаем туда /var:

root@user:/# mkfs.ext4 /dev/vg_var/lv_var
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 243712 4k blocks and 60928 inodes
Filesystem UUID: 8c2f1272-6484-4e8e-a27e-7ada7ea1da8c
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

root@user:/# mount /dev/vg_var/lv_var /mnt
root@user:/# cp -aR /var/* /mnt/

На всякий случай сохраняем содержимое старого var (или же можно его просто удалить):

root@user:/# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

Ну и монтируем новый var в каталог /var:

root@user:/# umount /mnt
root@user:/# mount /dev/vg_var/lv_var /var

Правим fstab для автоматического монтирования /var:

[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` \
 /var ext4 defaults 0 0" >> /etc/fstab


Презагружаемся (предварительно выйдя из chroot) в новый (уменьшенный root) и удалять
временную Volume Group:
root@user:/home/user# reboot

root@user:/home/user# lvremove /dev/vg_root/lv_root
Do you really want to remove and DISCARD active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed.

root@user:/home/user# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed

root@user:/home/user# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.

Проверяем структуру дисков:
root@user:/home/user# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   10G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0  8.2G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:6    0    8G  0 lvm  /
sdb                         8:16   0   10G  0 disk 
sdc                         8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0   252:1    0    4M  0 lvm  
│ └─vg_var-lv_var         252:5    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0  252:2    0  952M  0 lvm  
  └─vg_var-lv_var         252:5    0  952M  0 lvm  /var
sdd                         8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1   252:3    0    4M  0 lvm  
│ └─vg_var-lv_var         252:5    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1  252:4    0  952M  0 lvm  
  └─vg_var-lv_var         252:5    0  952M  0 lvm  /var
sde                         8:64   0    1G  0 disk 


### 3. Выделить том под /home.
Выделяем том под /home по тому же принципу что делали для /var:

root@user:/home/user# pvcreate /dev/sde
  Physical volume "/dev/sde" successfully created.

root@user:/home/user# vgcreate vg_home /dev/sde
  Volume group "vg_home" successfully created


root@user:/home/user# lvcreate -n LogVol_Home -L 1020m vg_home
  Logical volume "LogVol_Home" created.

root@user:/home/user# mkfs.ext4 /dev/vg_home/LogVol_Home
root@user:/home/user# mount /dev/vg_home/LogVol_Home /mnt/
root@user:/home/user# cp -aR /home/* /mnt/
root@user:/home/user# rm -rf /home/*
root@user:/home/user# umount /mnt
root@user:/home/user# mount /dev/vg_home/LogVol_Home /home/


### 4. Прописать монтирование в fstab. 
Правим fstab для автоматического монтирования /home:

root@user:/home/user# echo "`blkid | grep Home | awk '{print $2}'` \
 /home xfs defaults 0 0" >> /etc/fstab

Проверяем разделы:
root@user:/home/user# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   10G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0  8.2G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:6    0    8G  0 lvm  /
sdb                         8:16   0   10G  0 disk 
sdc                         8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0   252:1    0    4M  0 lvm  
│ └─vg_var-lv_var         252:5    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0  252:2    0  952M  0 lvm  
  └─vg_var-lv_var         252:5    0  952M  0 lvm  /var
sdd                         8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1   252:3    0    4M  0 lvm  
│ └─vg_var-lv_var         252:5    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1  252:4    0  952M  0 lvm  
  └─vg_var-lv_var         252:5    0  952M  0 lvm  /var
sde                         8:64   0    1G  0 disk 
└─vg_home-LogVol_Home     252:0    0 1020M  0 lvm  /home


### 5. Работа со снапшотами
Генерируем файлы в /home/:

root@user:/home/user# touch /home/file{1..20}


Увеличиваем диск:

root@user:/home/user# vgextend vg_home /dev/sdb
  Physical volume "/dev/sdb" successfully created.
  Volume group "vg_home" successfully extended
root@user:/home/user# lvresize -r -L +2G /dev/mapper/vg_home-LogVol_Home 
shell-init: error retrieving current directory: getcwd: cannot access parent directories: No such file or directory
  Size of logical volume vg_home/LogVol_Home changed from 1020.00 MiB (255 extents) to <3.00 GiB (767 extents).
  Logical volume vg_home/LogVol_Home successfully resized.
shell-init: error retrieving current directory: getcwd: cannot access parent directories: No such file or directory
resize2fs 1.47.0 (5-Feb-2023)
Filesystem at /dev/mapper/vg_home-LogVol_Home is mounted on /home; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/mapper/vg_home-LogVol_Home is now 785408 (4k) blocks long.

Снять снапшот:

root@user:/home/user# lvcreate -L 100MB -s -n home_snap \
 /dev/vg_home/LogVol_Home
  Logical volume "home_snap" created.

Удалить часть файлов:

root@user:/home/user# rm -f /home/file{11..20}

Проверяем:
root@user:/home/user# ll /home/
total 28
drwxr-xr-x  4 root root  4096 Apr 14 07:49 ./
drwxr-xr-x 23 root root  4096 Apr 10 05:51 ../
-rw-r--r--  1 root root     0 Apr 14 06:56 file1
-rw-r--r--  1 root root     0 Apr 14 06:56 file10
-rw-r--r--  1 root root     0 Apr 14 06:56 file2
-rw-r--r--  1 root root     0 Apr 14 06:56 file3
-rw-r--r--  1 root root     0 Apr 14 06:56 file4
-rw-r--r--  1 root root     0 Apr 14 06:56 file5
-rw-r--r--  1 root root     0 Apr 14 06:56 file6
-rw-r--r--  1 root root     0 Apr 14 06:56 file7
-rw-r--r--  1 root root     0 Apr 14 06:56 file8
-rw-r--r--  1 root root     0 Apr 14 06:56 file9
drwx------  2 root root 16384 Apr 14 06:51 lost+found/
drwxr-x---  5 user user  4096 Apr  1 18:32 user/

Процесс восстановления из снапшота:

root@user:/home/user# umount /home
root@user:/home/user# lvconvert --merge /dev/vg_home/home_snap
  Merging of volume vg_home/home_snap started.
  vg_home/LogVol_Home: Merged: 100.00%

root@user:/home/user# mount /dev/mapper/vg_home-LogVol_Home /home
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
root@user:/home/user# systemctl daemon-reload

[root@lvm ~]# ls -al /home
root@user:/home/user# ls -al /home
total 28
drwxr-xr-x  4 root root  4096 Apr 14 06:56 .
drwxr-xr-x 23 root root  4096 Apr 10 05:51 ..
-rw-r--r--  1 root root     0 Apr 14 06:56 file1
-rw-r--r--  1 root root     0 Apr 14 06:56 file10
-rw-r--r--  1 root root     0 Apr 14 06:56 file11
-rw-r--r--  1 root root     0 Apr 14 06:56 file12
-rw-r--r--  1 root root     0 Apr 14 06:56 file13
-rw-r--r--  1 root root     0 Apr 14 06:56 file14
-rw-r--r--  1 root root     0 Apr 14 06:56 file15
-rw-r--r--  1 root root     0 Apr 14 06:56 file16
-rw-r--r--  1 root root     0 Apr 14 06:56 file17
-rw-r--r--  1 root root     0 Apr 14 06:56 file18
-rw-r--r--  1 root root     0 Apr 14 06:56 file19
-rw-r--r--  1 root root     0 Apr 14 06:56 file2
-rw-r--r--  1 root root     0 Apr 14 06:56 file20
-rw-r--r--  1 root root     0 Apr 14 06:56 file3
-rw-r--r--  1 root root     0 Apr 14 06:56 file4
-rw-r--r--  1 root root     0 Apr 14 06:56 file5
-rw-r--r--  1 root root     0 Apr 14 06:56 file6
-rw-r--r--  1 root root     0 Apr 14 06:56 file7
-rw-r--r--  1 root root     0 Apr 14 06:56 file8
-rw-r--r--  1 root root     0 Apr 14 06:56 file9
drwx------  2 root root 16384 Apr 14 06:51 lost+found
drwxr-x---  5 user user  4096 Apr  1 18:32 user

Файлы успешно восстановлены с помощью снапшота.
