# Домашнее задание "LDAP"

## Цель
Научиться настраивать LDAP-сервер и подключать к нему LDAP-клиентов.

## Задание
Установить FreeIPA
Конфигурация клиента

---

## Выполнение задания
Для выполнения задания будет использоваться ОС Ubuntu 22.04.5 LTS для клиентских ВМ и
Rocky Linux 9.8 (Blue Onyx) для сервера.

Создаем на proxmox 3 виртуальные машины.
ipa.otus.lan: IP 192.168.1.45/24
client1.otus.lan: IP 192.168.1.129/24
client2.otus.lan: IP 192.168.1.114/24


Настроим hostname и /etc/hosts на всех ВМ
```bash
root@ipa:/home/user# sudo hostnamectl set-hostname ipa.otus.lan
root@client1:/home/user# sudo hostnamectl set-hostname client1.otus.lan
root@client2:/home/user# sudo hostnamectl set-hostname client2.otus.lan
```

На каждой из трёх машин /etc/hosts:
```bash
sudo nano /etc/hosts

127.0.0.1       localhost
127.0.1.1       ipa.otus.lan ipa

192.168.1.45   ipa.otus.lan ipa
192.168.1.129   client1.otus.lan client1
192.168.1.114   client2.otus.lan client2
--------------
127.0.0.1       localhost
127.0.1.1       client1.otus.lan client1

192.168.1.45   ipa.otus.lan ipa
192.168.1.129   client1.otus.lan client1
192.168.1.114   client2.otus.lan client2
----------
127.0.0.1       localhost
127.0.1.1       client2.otus.lan client2

192.168.1.45   ipa.otus.lan ipa
192.168.1.129   client1.otus.lan client1
192.168.1.114   client2.otus.lan client2
-------------
```
Проверка связанности:
```bash
[root@ipa ~]# ping -c 2 ipa.otus.lan
ping -c 2 client1.otus.lan
ping -c 2 client2.otus.lan
PING ipa.otus.lan (127.0.1.1) 56(84) bytes of data.
64 bytes from ipa.otus.lan (127.0.1.1): icmp_seq=1 ttl=64 time=0.048 ms
64 bytes from ipa.otus.lan (127.0.1.1): icmp_seq=2 ttl=64 time=0.020 ms

--- ipa.otus.lan ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1054ms
rtt min/avg/max/mdev = 0.020/0.034/0.048/0.014 ms
PING client1.otus.lan (192.168.1.129) 56(84) bytes of data.
64 bytes from client1.otus.lan (192.168.1.129): icmp_seq=1 ttl=64 time=0.200 ms
64 bytes from client1.otus.lan (192.168.1.129): icmp_seq=2 ttl=64 time=0.303 ms

--- client1.otus.lan ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.200/0.251/0.303/0.051 ms
PING client2.otus.lan (192.168.1.114) 56(84) bytes of data.
64 bytes from client2.otus.lan (192.168.1.114): icmp_seq=1 ttl=64 time=0.202 ms
64 bytes from client2.otus.lan (192.168.1.114): icmp_seq=2 ttl=64 time=0.226 ms

--- client2.otus.lan ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.202/0.214/0.226/0.012 ms

root@client1:/home/user# ping -c 2 ipa.otus.lan
ping -c 2 client1.otus.lan
ping -c 2 client2.otus.lan
PING ipa.otus.lan (192.168.1.45) 56(84) bytes of data.
64 bytes from ipa.otus.lan (192.168.1.45): icmp_seq=1 ttl=64 time=0.270 ms
64 bytes from ipa.otus.lan (192.168.1.45): icmp_seq=2 ttl=64 time=0.310 ms

--- ipa.otus.lan ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.270/0.290/0.310/0.020 ms
PING client1.otus.lan (127.0.1.1) 56(84) bytes of data.
64 bytes from client1.otus.lan (127.0.1.1): icmp_seq=1 ttl=64 time=0.014 ms
64 bytes from client1.otus.lan (127.0.1.1): icmp_seq=2 ttl=64 time=0.020 ms

--- client1.otus.lan ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.014/0.017/0.020/0.003 ms
PING client2.otus.lan (192.168.1.114) 56(84) bytes of data.
64 bytes from client2.otus.lan (192.168.1.114): icmp_seq=1 ttl=64 time=0.249 ms
64 bytes from client2.otus.lan (192.168.1.114): icmp_seq=2 ttl=64 time=0.270 ms

--- client2.otus.lan ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.249/0.259/0.270/0.010 ms

root@client2:/home/user# ping -c 2 ipa.otus.lan
ping -c 2 client1.otus.lan
ping -c 2 client2.otus.lan
PING ipa.otus.lan (192.168.1.45) 56(84) bytes of data.
64 bytes from ipa.otus.lan (192.168.1.45): icmp_seq=1 ttl=64 time=0.369 ms
64 bytes from ipa.otus.lan (192.168.1.45): icmp_seq=2 ttl=64 time=0.306 ms

--- ipa.otus.lan ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1030ms
rtt min/avg/max/mdev = 0.306/0.337/0.369/0.031 ms
PING client1.otus.lan (192.168.1.129) 56(84) bytes of data.
64 bytes from client1.otus.lan (192.168.1.129): icmp_seq=1 ttl=64 time=0.206 ms
64 bytes from client1.otus.lan (192.168.1.129): icmp_seq=2 ttl=64 time=0.273 ms

--- client1.otus.lan ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1021ms
rtt min/avg/max/mdev = 0.206/0.239/0.273/0.033 ms
PING client2.otus.lan (127.0.1.1) 56(84) bytes of data.
64 bytes from client2.otus.lan (127.0.1.1): icmp_seq=1 ttl=64 time=0.016 ms
64 bytes from client2.otus.lan (127.0.1.1): icmp_seq=2 ttl=64 time=0.019 ms

--- client2.otus.lan ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.016/0.017/0.019/0.001 ms
```

Установим базовые пакеты на всех ВМ
```bash
[root@ipa ~]# sudo yum update
[root@ipa ~]# sudo yum install -y vim chrony
```
Настроим chrony на сервере
```bash
[root@ipa ~]# sudo nano /etc/chrony.conf

allow 192.168.1.0/24
local stratum 10


[root@ipa ~]# firewall-cmd --permanent --add-service=ntp
success
[root@ipa ~]# firewall-cmd --reload
success
[root@ipa ~]# sudo firewall-cmd --list-services
cockpit dhcpv6-client ntp ssh
[root@ipa ~]# sudo firewall-cmd --list-services
cockpit dhcpv6-client ntp ssh
[root@ipa ~]# sudo ss -ulnp | grep chronyd
UNCONN 0      0          127.0.0.1:323       0.0.0.0:*    users:(("chronyd",pid=693,fd=5))
UNCONN 0      0              [::1]:323          [::]:*    users:(("chronyd",pid=693,fd=6))


[root@ipa ~]# sudo systemctl restart chronyd
[root@ipa ~]# sudo systemctl enable chronyd


[root@ipa ~]# sudo systemctl enable chronyd
[root@ipa ~]# chronyc tracking
Reference ID    : 33FA44C6 (51.250.68.198)
Stratum         : 3
Ref time (UTC)  : Fri Sep 25 07:00:58 2026
System time     : 0.000000082 seconds fast of NTP time
Last offset     : -0.000365912 seconds
RMS offset      : 0.000365912 seconds
Frequency       : 0.786 ppm slow
Residual freq   : -62.505 ppm
Skew            : 1.168 ppm
Root delay      : 0.023693288 seconds
Root dispersion : 0.002968093 seconds
Update interval : 1.6 seconds
Leap status     : Normal

```
Настроим chrony на клиентах
```bash
sudo nano /etc/chrony/chrony.conf

# pool ntp.ubuntu.com        iburst maxsources 4
# pool 0.ubuntu.pool.ntp.org iburst maxsources 1
# pool 1.ubuntu.pool.ntp.org iburst maxsources 1

server ipa.otus.lan iburst

sudo systemctl restart chrony

root@client1:/home/user# chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* ipa.otus.lan                  3   6    17     1  +3902ns[+6197ns] +/-   26ms

root@client2:/home/user# chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* ipa.otus.lan                  3   6    17     0    +56us[ +488us] +/-   26ms

```
Установим FreeIPA-сервер

```bash
[root@ipa ~]# sudo dnf install -y ipa-server ipa-server-dns
```
Убедидимся, что firewalld запущен
```bash
[root@ipa ~]# sudo systemctl status firewalld --no-pager
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-09-25 08:49:50 CEST; 1h 7min ago
       Docs: man:firewalld(1)
   Main PID: 690 (firewalld)
      Tasks: 4 (limit: 4606)
     Memory: 29.8M (peak: 46.3M)
        CPU: 711ms
     CGroup: /system.slice/firewalld.service
             └─690 /usr/bin/python3 -s /usr/sbin/firewalld --nofo…
Sep 25 08:49:47 ipa systemd[1]: Starting firewalld - dynamic f…...Sep 25 08:49:50 ipa systemd[1]: Started firewalld - dynamic fi…on.Hint: Some lines were ellipsized, use -l to show in full.
```

Запустим установку FreeIPA-сервера
```bash
sudo ipa-server-install --unattended \
  --hostname=ipa.otus.lan \
  --domain=otus.lan \
  --realm=OTUS.LAN \
  --ds-password='DirectoryManager123' \
  --admin-password='FreeIPAAdmin123' \
  --setup-dns \
  --no-forwarders \
  --no-ntp


...
The ipa-server-install command was successful
...

Параметры:
--hostname=ipa.otus.lan — FQDN сервера (должен совпадать с hostname -f).
--domain=otus.lan — DNS-домен.
--realm=OTUS.LAN — Kerberos-realm (верхний регистр).
--ds-password — пароль Directory Manager (≥ 8 символов).
--admin-password — пароль FreeIPA admin (≥ 8 символов).
--setup-dns — поднять встроенный BIND.
--no-forwarders — не пересылать внешние DNS-запросы (лаборатория изолирована).
--no-ntp — мы уже настроили chrony.
```

Откроем порты FreeIPA в firewalld
```bash
[root@ipa ~]# sudo firewall-cmd --permanent --add-service=freeipa-ldap
sudo firewall-cmd --permanent --add-service=freeipa-ldaps
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=kerberos
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
success
success
success
success
success
cockpit dhcpv6-client dns freeipa-ldap freeipa-ldaps kerberos ntp ssh
```

Настроим DNS на Ubuntu-клиентах
```bash
sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
      nameservers:
        addresses: [192.168.1.45]
        search: [otus.lan]

sudo netplan generate
sudo netplan apply
```
Проверяем, что клиенты резолвят сервер и SRV-записи через DNS (а не через /etc/hosts):

```bash
root@client1:/home/user# dig ipa.otus.lan +short
dig _kerberos._udp.otus.lan SRV +short
dig _ldap._tcp.otus.lan SRV +short
192.168.1.45
0 100 88 ipa.otus.lan.
0 100 389 ipa.otus.lan.

root@client2:/home/user# dig ipa.otus.lan +short
dig _kerberos._udp.otus.lan SRV +short
dig _ldap._tcp.otus.lan SRV +short
192.168.1.45
0 100 88 ipa.otus.lan.
0 100 389 ipa.otus.lan.
```

Установка клиента FreeIPA
```bash
sudo apt update
sudo apt install -y freeipa-client oddjob-mkhomedir
```

Подключим клиентов к домену
```bash
sudo ipa-client-install --unattended \
  --domain=otus.lan \
  --server=ipa.otus.lan \
  --realm=OTUS.LAN \
  --mkhomedir \
  --no-ntp \
  -p admin -w 'FreeIPAAdmin123'

....
Client configuration complete.
The ipa-client-install command was successful

```

Проверка после подключения
```bash
root@client1:/home/user# kinit admin
Password for admin@OTUS.LAN: 
root@client1:/home/user# 

root@client1:/home/user# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: admin@OTUS.LAN

Valid starting       Expires              Service principal
09/25/2026 10:42:45  09/26/2026 10:02:15  krbtgt/OTUS.LAN@OTUS.LANroot@client1:/home/user# 


root@client1:/home/user# sudo systemctl status sssd --no-pager
● sssd.service - System Security Services Daemon
     Loaded: loaded (/lib/systemd/system/sssd.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2026-09-25 10:40:42 UTC; 3min 44s ago
   Main PID: 7196 (sssd)
      Tasks: 7 (limit: 1008)
     Memory: 53.6M
        CPU: 165ms
     CGroup: /system.slice/sssd.service
             ├─7196 /usr/sbin/sssd -i --logger=files
             ├─7197 /usr/libexec/sssd/sssd_be --domain otus.lan -…             ├─7198 /usr/libexec/sssd/sssd_nss --uid 0 --gid 0 --…             ├─7199 /usr/libexec/sssd/sssd_pam --uid 0 --gid 0 --…             ├─7200 /usr/libexec/sssd/sssd_ssh --uid 0 --gid 0 --…             ├─7201 /usr/libexec/sssd/sssd_sudo --uid 0 --gid 0 -…             └─7202 /usr/libexec/sssd/sssd_pac --uid 0 --gid 0 --…
Sep 25 10:40:42 client1.otus.lan sssd_sudo[7201]: Starting up
Sep 25 10:40:42 client1.otus.lan sssd_pam[7199]: Starting up
Sep 25 10:40:42 client1.otus.lan sssd_ssh[7200]: Starting up
Sep 25 10:40:42 client1.otus.lan sssd_nss[7198]: Starting up
Sep 25 10:40:42 client1.otus.lan sssd_be[7197]: GSSAPI client s… 1Sep 25 10:40:42 client1.otus.lan sssd_be[7197]: GSSAPI client s… 1Sep 25 10:40:42 client1.otus.lan sssd_pac[7202]: Starting up
Sep 25 10:40:42 client1.otus.lan sssd_be[7197]: GSSAPI client s… 1Sep 25 10:40:42 client1.otus.lan sssd_be[7197]: GSSAPI client s… 2Sep 25 10:40:42 client1.otus.lan systemd[1]: Started System Sec…n.Hint: Some lines were ellipsized, use -l to show in full.


root@client1:/home/user# dig _kerberos._udp.otus.lan SRV +short
0 100 88 ipa.otus.lan.
root@client1:/home/user# dig _ldap._tcp.otus.lan SRV +short
0 100 389 ipa.otus.lan.

root@client2:/home/user# kinit admin
Password for admin@OTUS.LAN: 
root@client2:/home/user# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: admin@OTUS.LAN

Valid starting       Expires              Service principal
09/25/2026 10:48:31  09/26/2026 10:12:51  krbtgt/OTUS.LAN@OTUS.LANroot@client2:/home/user# id admin@otus.lan
uid=412400000(admin) gid=412400000(admins) groups=412400000(admins)

root@client2:/home/user# sudo systemctl status sssd --no-pager
● sssd.service - System Security Services Daemon
     Loaded: loaded (/lib/systemd/system/sssd.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2026-09-25 10:40:51 UTC; 8min ago
   Main PID: 7113 (sssd)
      Tasks: 7 (limit: 1008)
     Memory: 52.6M
        CPU: 168ms
     CGroup: /system.slice/sssd.service
             ├─7113 /usr/sbin/sssd -i --logger=files
             ├─7114 /usr/libexec/sssd/sssd_be --domain otus.lan -…             ├─7115 /usr/libexec/sssd/sssd_nss --uid 0 --gid 0 --…             ├─7116 /usr/libexec/sssd/sssd_pam --uid 0 --gid 0 --…             ├─7117 /usr/libexec/sssd/sssd_ssh --uid 0 --gid 0 --…             ├─7118 /usr/libexec/sssd/sssd_sudo --uid 0 --gid 0 -…             └─7119 /usr/libexec/sssd/sssd_pac --uid 0 --gid 0 --…
Sep 25 10:40:50 client2.otus.lan sssd_ssh[7117]: Starting up
Sep 25 10:40:50 client2.otus.lan sssd_be[7114]: GSSAPI client s… 1Sep 25 10:40:50 client2.otus.lan sssd_be[7114]: GSSAPI client s… 1Sep 25 10:40:50 client2.otus.lan sssd_sudo[7118]: Starting up
Sep 25 10:40:50 client2.otus.lan sssd_pam[7116]: Starting up
Sep 25 10:40:50 client2.otus.lan sssd_be[7114]: GSSAPI client s… 1Sep 25 10:40:50 client2.otus.lan sssd_nss[7115]: Starting up
Sep 25 10:40:50 client2.otus.lan sssd_pac[7119]: Starting up
Sep 25 10:40:50 client2.otus.lan sssd_be[7114]: GSSAPI client s… 2Sep 25 10:40:51 client2.otus.lan systemd[1]: Started System Sec…n.Hint: Some lines were ellipsized, use -l to show in full.
```
Создадим тестового пользователя на сервере
```bash
[root@ipa ~]# kinit admin
Password for admin@OTUS.LAN: # вводим FreeIPAAdmin123
[root@ipa ~]# ipa user-add otus-user --first=Otus --last=User --password
Password: # вводим OtusUser123
Enter Password again to verify: 
----------------------
Added user "otus-user"
----------------------
  User login: otus-user
  First name: Otus
  Last name: User
  Full name: Otus User
  Display name: Otus User
  Initials: OU
  Home directory: /home/otus-user
  GECOS: Otus User
  Login shell: /bin/sh
  Principal name: otus-user@OTUS.LAN
  Principal alias: otus-user@OTUS.LAN
  User password expiration: 20260925105155Z
  Email address: otus-user@otus.lan
  UID: 412400003
  GID: 412400003
  Password: True
  Member of groups: ipausers
  Kerberos keys available: True


[root@ipa ~]# ipa user-find otus-user
--------------
1 user matched
--------------
  User login: otus-user
  First name: Otus
  Last name: User
  Home directory: /home/otus-user
  Login shell: /bin/sh
  Principal name: otus-user@OTUS.LAN
  Principal alias: otus-user@OTUS.LAN
  Email address: otus-user@otus.lan
  UID: 412400003
  GID: 412400003
  Account disabled: False
----------------------------
Number of entries returned 1
----------------------------
```

Проверяем вход пользователя на client1
```bash
root@client1:/home/user# kinit admin
Password for admin@OTUS.LAN: 
root@client1:/home/user# ipa user-add otus-user --first=Otus --last=User --password
Password: 
Enter Password again to verify: 
ipa: ERROR: Could not get Password interactively
root@client1:/home/user# 
root@client1:/home/user# 
root@client1:/home/user# 
root@client1:/home/user# kdestroy
root@client1:/home/user# kinit otus-user
Password for otus-user@OTUS.LAN: 
Password expired.  You must change it now.
Enter new password: 
Enter it again:


root@client1:/home/user# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: otus-user@OTUS.LAN

Valid starting       Expires              Service principal
09/25/2026 10:56:50  09/26/2026 10:39:37  krbtgt/OTUS.LAN@OTUS.LAN


root@client1:/home/user# id otus-user
uid=412400003(otus-user) gid=412400003(otus-user) groups=412400003(otus-user)
root@client1:/home/user# getent passwd otus-user
otus-user:*:412400003:412400003:Otus User:/home/otus-user:/bin/sh

root@client1:/home/user# ssh otus-user@client2.otus.lan
Creating directory '/home/otus-user'.
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-186-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Sep 25 10:58:35 AM UTC 2026

  System load:  0.08              Processes:              116
  Usage of /:   56.2% of 9.75GB   Users logged in:        1
  Memory usage: 26%               IPv4 address for ens18: 192.168.1.114
  Swap usage:   1%


Expanded Security Maintenance for Applications is not enabled.

95 updates can be applied immediately.
86 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '24.04.5 LTS' available.
Run 'do-release-upgrade' to upgrade to it.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in theindividual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

$ 

$ whoami
pwd
id
exit
otus-user
$ /home/otus-user
$ uid=412400003(otus-user) gid=412400003(otus-user) groups=412400003(otus-user)
```

Проверяем вход пользователя на client2
```bash
root@client2:/home/user# kdestroy
root@client2:/home/user# kinit otus-user
Password for otus-user@OTUS.LAN: 
root@client2:/home/user# ssh otus-user@client1.otus.lan
Creating directory '/home/otus-user'.
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-186-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Sep 25 11:07:52 AM UTC 2026

  System load:  0.08              Processes:              117
  Usage of /:   54.3% of 9.75GB   Users logged in:        1
  Memory usage: 25%               IPv4 address for ens18: 192.168.1.129
  Swap usage:   1%

 * Canonical Workshop gives developers fast, composable, reproducible, and
   secure developer environments that are perfect for agentic workflows.

   https://ubuntu.com/workshop

Expanded Security Maintenance for Applications is not enabled.

95 updates can be applied immediately.
86 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '24.04.5 LTS' available.
Run 'do-release-upgrade' to upgrade to it.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in theindividual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

$ whoami
pwd
id
exotus-user
$ /home/otus-user
$ uid=412400003(otus-user) gid=412400003(otus-user) groups=412400003(otus-user)
$ it
Connection to client1.otus.lan closed.
```

Всё, что нужно, работает:

FreeIPA-сервер установлен и настроен.

Оба клиента подключены к домену, SSSD работает, Kerberos выдаёт билеты.

Пользователь otus-user создан и успешно логинится по SSH между client1 и client2 в обе стороны.

Домашняя директория /home/otus-user автоматически создаётся при первом входе благодаря --mkhomedir.
