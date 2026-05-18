# Домашнее задание "Работа с NFS"

## Задание

- Запустить 2 виртуальных машины (сервер NFS и клиента);
- На сервере NFS должна быть подготовлена и экспортирована директория; 
- В экспортированной директории должна быть поддиректория с именем upload с правами на запись в неё; 
- Экспортированная директория должна автоматически монтироваться на клиенте при старте виртуальной машины (systemd, autofs или fstab — любым способом);
- Монтирование и работа NFS на клиенте должна быть организована с использованием NFSv3.

---

## Выполнение домашнего задания

Созданы две тестовые виртуальные машины с сетевыми интерфейсами, которые позволяют связь между ними. 
Далее будем называть ВМ с NFS сервером srv (IP 192.168.1.95), а ВМ с клиентом client (IP 192.168.1.103).

```bash 
root@client:/home/user# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp0s3           UP             192.168.1.103/24 metric 100 fe80::a00:27ff:fec5:2ef6/64 


root@srv:/home/srv# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp0s3           UP             192.168.1.95/24 metric 100 fe80::a00:27ff:feca:5e76/64 
```

### Настраиваем сервер NFS 
Заходим на сервер и установим сервер NFS:
```bash 
root@srv:/home/srv# apt install nfs-kernel-server
```

Настройки сервера находятся в файле /etc/nfs.conf 
Проверяем наличие слушающих портов 2049/udp, 2049/tcp,111/udp, 111/tcp (не все они будут использоваться далее,  но их наличие сигнализирует о том, что необходимые сервисы готовы принимать внешние подключения):
root@srv:/home/srv# ss -tulpn

<details>

```bash
Netid      State       Recv-Q      Send-Q                 Local Address:Port              Peer Address:Port      Process                                                          
udp        UNCONN      0           0                            0.0.0.0:43471                  0.0.0.0:*          users:(("rpc.statd",pid=1854,fd=8))                             
udp        UNCONN      0           0                            0.0.0.0:57907                  0.0.0.0:*          users:(("rpc.mountd",pid=1861,fd=8))                            
udp        UNCONN      0           0                         127.0.0.54:53                     0.0.0.0:*          users:(("systemd-resolve",pid=486,fd=16))                       
udp        UNCONN      0           0                      127.0.0.53%lo:53                     0.0.0.0:*          users:(("systemd-resolve",pid=486,fd=14))                       
udp        UNCONN      0           0                192.168.1.95%enp0s3:68                     0.0.0.0:*          users:(("systemd-network",pid=475,fd=11))                       
udp        UNCONN      0           0                            0.0.0.0:111                    0.0.0.0:*          users:(("rpcbind",pid=1349,fd=5),("systemd",pid=1,fd=106))      
udp        UNCONN      0           0                            0.0.0.0:48253                  0.0.0.0:*                                                                          
udp        UNCONN      0           0                            0.0.0.0:35969                  0.0.0.0:*          users:(("rpc.mountd",pid=1861,fd=12))                           
udp        UNCONN      0           0                          127.0.0.1:758                    0.0.0.0:*          users:(("rpc.statd",pid=1854,fd=5))                             
udp        UNCONN      0           0                            0.0.0.0:54556                  0.0.0.0:*          users:(("rpc.mountd",pid=1861,fd=4))                            
udp        UNCONN      0           0                               [::]:56281                     [::]:*          users:(("rpc.statd",pid=1854,fd=10))                            
udp        UNCONN      0           0                               [::]:111                       [::]:*          users:(("rpcbind",pid=1349,fd=7),("systemd",pid=1,fd=108))      
udp        UNCONN      0           0                               [::]:51552                     [::]:*          users:(("rpc.mountd",pid=1861,fd=6))                            
udp        UNCONN      0           0                               [::]:43379                     [::]:*          users:(("rpc.mountd",pid=1861,fd=14))                           
udp        UNCONN      0           0                               [::]:36729                     [::]:*                                                                          
udp        UNCONN      0           0                               [::]:43389                     [::]:*          users:(("rpc.mountd",pid=1861,fd=10))                           
tcp        LISTEN      0           64                           0.0.0.0:36731                  0.0.0.0:*                                                                          
tcp        LISTEN      0           4096                         0.0.0.0:51041                  0.0.0.0:*          users:(("rpc.mountd",pid=1861,fd=9))                            
tcp        LISTEN      0           4096                         0.0.0.0:41405                  0.0.0.0:*          users:(("rpc.mountd",pid=1861,fd=5))                            
tcp        LISTEN      0           4096                      127.0.0.54:53                     0.0.0.0:*          users:(("systemd-resolve",pid=486,fd=17))                       
tcp        LISTEN      0           4096                         0.0.0.0:111                    0.0.0.0:*          users:(("rpcbind",pid=1349,fd=4),("systemd",pid=1,fd=105))      
tcp        LISTEN      0           4096                         0.0.0.0:22                     0.0.0.0:*          users:(("sshd",pid=1041,fd=3),("systemd",pid=1,fd=124))         
tcp        LISTEN      0           64                           0.0.0.0:2049                   0.0.0.0:*                                                                          
tcp        LISTEN      0           4096                         0.0.0.0:46121                  0.0.0.0:*          users:(("rpc.statd",pid=1854,fd=9))                             
tcp        LISTEN      0           4096                         0.0.0.0:33943                  0.0.0.0:*          users:(("rpc.mountd",pid=1861,fd=13))                           
tcp        LISTEN      0           4096                   127.0.0.53%lo:53                     0.0.0.0:*          users:(("systemd-resolve",pid=486,fd=15))                       
tcp        LISTEN      0           4096                            [::]:33065                     [::]:*          users:(("rpc.mountd",pid=1861,fd=15))                           
tcp        LISTEN      0           64                              [::]:33769                     [::]:*                                                                          
tcp        LISTEN      0           4096                            [::]:39025                     [::]:*          users:(("rpc.mountd",pid=1861,fd=11))                           
tcp        LISTEN      0           4096                            [::]:111                       [::]:*          users:(("rpcbind",pid=1349,fd=6),("systemd",pid=1,fd=107))      
tcp        LISTEN      0           4096                            [::]:22                        [::]:*          users:(("sshd",pid=1041,fd=4),("systemd",pid=1,fd=125))         
tcp        LISTEN      0           64                              [::]:2049                      [::]:*                                                                          
tcp        LISTEN      0           4096                            [::]:44189                     [::]:*          users:(("rpc.statd",pid=1854,fd=11))                            
tcp        LISTEN      0           4096                            [::]:53385                     [::]:*          users:(("rpc.mountd",pid=1861,fd=7))            

```
</details>

Создаём и настраиваем директорию, которая будет экспортирована в будущем: 
```bash
root@srv:/home/srv# mkdir -p /srv/share/upload
root@srv:/home/srv# chown -R nobody:nogroup /srv/share
root@srv:/home/srv# chmod 0777 /srv/share/upload
```

Cоздаём в файле /etc/exports структуру, которая позволит экспортировать ранее созданную директорию:
```bash
root@srv:/home/srv# cat << EOF > /etc/exports
> /srv/share 192.168.1.103/32(rw,sync,root_squash)
> EOF
```

Экспортируем ранее созданную директорию:
```bash
root@srv:/home/srv# exportfs -r
exportfs: /etc/exports [1]: Neither 'subtree_check' or 'no_subtree_check' specified for export "192.168.1.103/32:/srv/share".
  Assuming default behaviour ('no_subtree_check').
  NOTE: this default has changed since nfs-utils version 1.0.x
```
Проверяем экспортированную директорию следующей командой:
```bash
root@srv:/home/srv# exportfs -s
/srv/share  192.168.1.103/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```
### Настраиваем клиент NFS.
Заходим на сервер с клиентом.
Установим пакет с NFS-клиентом
```bash
root@client:/home/user# sudo apt install nfs-common
```

Добавляем в /etc/fstab строку 
```bash
root@client:/home/user# echo "192.168.1.95:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0" >> /etc/fstab
```
и выполняем команды:
```bash
root@client:/home/user# systemctl daemon-reload
root@client:/home/user# systemctl restart remote-fs.target
```

В данном случае происходит автоматическая генерация systemd units в каталоге /run/systemd/generator/, которые производят монтирование при первом обращении к каталогу /mnt/.
Заходим в директорию /mnt/ и проверяем успешность монтирования:
```bash
root@client:/mnt# mount | grep mnt 
systemd-1 on /mnt type autofs (rw,relatime,fd=70,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=16513)
192.168.1.95:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.1.95,mountvers=3,mountport=35969,mountproto=udp,local_lock=none,addr=192.168.1.95)
```
В параметрах видим `vers=3`, что соответствует NFSv3, как того требует задание.

Проверка работоспособности 
Заходим на сервер. 
Заходим в каталог /srv/share/upload. Создаём тестовый файл check_file.
```bash
root@srv:/home/srv# cd /srv/share/upload/
root@srv:/srv/share/upload# touch check_file
root@srv:/srv/share/upload# ll
total 8
drwxrwxrwx 2 nobody nogroup 4096 May 18 06:16 ./
drwxr-xr-x 3 nobody nogroup 4096 May 18 05:56 ../
-rw-r--r-- 1 root   root       0 May 18 06:16 check_file
```
Заходим на клиент.
Заходим в каталог /mnt/upload. Проверяем наличие ранее созданного файла. Создаём тестовый файл touch client_file. Проверяем, что файл успешно создан.
```bash
root@client:/mnt# cd /mnt/upload/
root@client:/mnt/upload# touch client_file
root@client:/mnt/upload# ll
total 8
drwxrwxrwx 2 nobody nogroup 4096 May 18 06:18 ./
drwxr-xr-x 3 nobody nogroup 4096 May 18 05:56 ../
-rw-r--r-- 1 root   root       0 May 18 06:16 check_file
-rw-r--r-- 1 nobody nogroup    0 May 18 06:18 client_file
```

Проверяем клиент: 
-перезагружаем клиент;
-заходим на клиент;
-заходим в каталог /mnt/upload;
-проверяем наличие ранее созданных файлов.
```bash
root@client:/mnt/upload# reboot
Broadcast message from root@client on pts/1 (Mon 2026-05-18 06:20:27 UTC):
The system will reboot now!

root@client:/home/user# cd /mnt/upload/
root@client:/mnt/upload# ll
total 8
drwxrwxrwx 2 nobody nogroup 4096 May 18 06:18 ./
drwxr-xr-x 3 nobody nogroup 4096 May 18 05:56 ../
-rw-r--r-- 1 root   root       0 May 18 06:16 check_file
-rw-r--r-- 1 nobody nogroup    0 May 18 06:18 client_file
```

Проверяем сервер: 
-заходим на сервер в отдельном окне терминала;
-перезагружаем сервер;
-заходим на сервер;
-проверяем наличие файлов в каталоге /srv/share/upload/;
-проверяем экспорты exportfs -s;
-проверяем работу RPC showmount -a 192.168.1.95.
 
```bash
root@srv:/srv/share/upload# reboot 
Broadcast message from root@srv on pts/1 (Mon 2026-05-18 06:22:43 UTC):
The system will reboot now!

root@srv:/home/srv# ll /srv/share/upload/
total 8
drwxrwxrwx 2 nobody nogroup 4096 May 18 06:18 ./
drwxr-xr-x 3 nobody nogroup 4096 May 18 05:56 ../
-rw-r--r-- 1 root   root       0 May 18 06:16 check_file
-rw-r--r-- 1 nobody nogroup    0 May 18 06:18 client_file

root@srv:/home/srv# exportfs -s
/srv/share  192.168.1.103/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

root@srv:/home/srv# showmount -a 192.168.1.95
All mount points on 192.168.1.95:
192.168.1.103:/srv/share
```

Проверяем клиент: 
-возвращаемся на клиент;
-перезагружаем клиент;
-заходим на клиент;
-проверяем работу RPC showmount -a 192.168.50.10;
-заходим в каталог /mnt/upload;
-проверяем статус монтирования mount | grep mnt;
-проверяем наличие ранее созданных файлов;
-создаём тестовый файл final_check;
-проверяем, что файл успешно создан.

```bash
root@client:/mnt/upload# reboot 
Broadcast message from root@client on pts/1 (Mon 2026-05-18 06:27:38 UTC):
The system will reboot now!

root@client:/home/user# showmount -a 192.168.1.95
All mount points on 192.168.1.95:
root@client:/home/user# cd /mnt/upload/
root@client:/mnt/upload# ll
total 8
drwxrwxrwx 2 nobody nogroup 4096 May 18 06:18 ./
drwxr-xr-x 3 nobody nogroup 4096 May 18 05:56 ../
-rw-r--r-- 1 root   root       0 May 18 06:16 check_file
-rw-r--r-- 1 nobody nogroup    0 May 18 06:18 client_file
root@client:/mnt/upload# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=58,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=4683)
192.168.1.95:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.1.95,mountvers=3,mountport=34862,mountproto=udp,local_lock=none,addr=192.168.1.95)
root@client:/mnt/upload# touch final_check
root@client:/mnt/upload# ll
total 8
drwxrwxrwx 2 nobody nogroup 4096 May 18 06:31 ./
drwxr-xr-x 3 nobody nogroup 4096 May 18 05:56 ../
-rw-r--r-- 1 root   root       0 May 18 06:16 check_file
-rw-r--r-- 1 nobody nogroup    0 May 18 06:18 client_file
-rw-r--r-- 1 nobody nogroup    0 May 18 06:31 final_check
```

Вышеуказанные проверки прошли успешно, это значит, что демонстрационный стенд работоспособен и готов к работе. 