# Домашнее задание "VPN"

## Цель
Создать домашнюю сетевую лабораторию. Научится настраивать VPN-сервер в Linux-based системах.

## Задание
1)Настроить VPN между двумя ВМ в tun/tap режимах, замерить скорость в туннелях, сделать вывод об отличающихся показателях
2)Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на ВМ
---

## Выполнение задания
Для выполнения задания будет использоваться ОС Ubuntu 22.04.5 LTS

Создаем на proxmox 2 виртуальные машины server и client.
server: IP 192.168.1.78/24
client: IP 192.168.1.104/24

Установим  необходимые пакеты на обе ВМ

```bash
apt update
apt install -y openvpn iperf3
modprobe tun
echo 1 > /proc/sys/net/ipv4/ip_forward
ls -l /dev/net/tun
systemctl disable --now openvpn.service
```


# Настроим TAP на сервере (192.168.1.78)
```bash
openvpn --genkey secret /etc/openvpn/static.key
```

Проверяем, что ключ создан:

```bash
ls -l /etc/openvpn/static.key
```

Создаем конфиг:
```bash
cat > /etc/openvpn/server.conf <<'EOF'
dev tap
ifconfig 10.10.10.1 255.255.255.0
topology subnet
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
EOF
```
Создаем systemd-юнит:
```bash
cat > /etc/systemd/system/openvpn@.service <<'EOF'
[Unit]
Description=OpenVPN Tunneling Application On %I
After=network.target

[Service]
Type=notify
PrivateTmp=true
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf

[Install]
WantedBy=multi-user.target
EOF
```

Запускаем:
```bash
root@server:/home/user# systemctl daemon-reload
root@server:/home/user# systemctl start openvpn@server
root@server:/home/user# systemctl enable openvpn@server
Created symlink /etc/systemd/system/multi-user.target.wants/openvpn@server.service → /etc/systemd/system/openvpn@.service.
root@server:/home/user# systemctl status openvpn@server --no-pager
● openvpn@server.service - OpenVPN Tunneling Application On server
     Loaded: loaded (/etc/systemd/system/openvpn@.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-09-22 06:19:14 UTC; 9s ago
   Main PID: 1955 (openvpn)
     Status: "Pre-connection initialization successful"
      Tasks: 1 (limit: 1008)
     Memory: 1.6M
        CPU: 7ms
     CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
             └─1955 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf

Sep 22 06:19:14 server systemd[1]: Starting OpenVPN Tunneling Application On server...
Sep 22 06:19:14 server systemd[1]: Started OpenVPN Tunneling Application On server.
Sep 22 06:19:14 server openvpn[1955]: 2026-09-22 06:19:14 WARNING: Compression for r… set.Hint: Some lines were ellipsized, use -l to show in full.
```
Проверяем интерфейс:
```bash
root@server:/home/user# ip -br a | grep tap
tap0             UNKNOWN        10.10.10.1/24 fe80::4c4c:25ff:fef0:161a/64 
```

# Настройка TAP на клиенте (192.168.1.104)
Скопируем ключ с сервера:
```bash
root@server:/home/user# cp /etc/openvpn/static.key /tmp/static.key
chmod 644 /tmp/static.key
ls -l /tmp/static.key
-rw-r--r-- 1 root root 636 Sep 22 06:24 /tmp/static.key

root@client:/home/user# scp user@192.168.1.78:/tmp/static.key /etc/openvpn/static.key
chmod 600 /etc/openvpn/static.key
ls -l /etc/openvpn/static.key
user@192.168.1.78's password: 
static.key                                              100%  636     1.1MB/s   00:00    
-rw------- 1 root root 636 Sep 22 06:24 /etc/openvpn/static.key
```

Создаем конфиг:
```bash
cat > /etc/openvpn/server.conf <<'EOF'
dev tap
remote 192.168.1.78
ifconfig 10.10.10.2 255.255.255.0
topology subnet
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
EOF
```

Создаем такой же systemd-юнит:
```bash
cat > /etc/systemd/system/openvpn@.service <<'EOF'
[Unit]
Description=OpenVPN Tunneling Application On %I
After=network.target

[Service]
Type=notify
PrivateTmp=true
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf

[Install]
WantedBy=multi-user.target
EOF
```

Запускаем:
```bash
root@client:/home/user# systemctl daemon-reload
root@client:/home/user# systemctl start openvpn@server
root@client:/home/user# systemctl enable openvpn@server
Created symlink /etc/systemd/system/multi-user.target.wants/openvpn@server.service → /etc/systemd/system/openvpn@.service.
root@client:/home/user# systemctl status openvpn@server --no-pager
● openvpn@server.service - OpenVPN Tunneling Application On server
     Loaded: loaded (/etc/systemd/system/openvpn@.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-09-22 06:28:29 UTC; 7s ago
   Main PID: 1935 (openvpn)
     Status: "Pre-connection initialization successful"
      Tasks: 1 (limit: 1008)
     Memory: 2.7M
        CPU: 8ms
     CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
             └─1935 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf

Sep 22 06:28:29 client systemd[1]: Starting OpenVPN Tunneling Application On server...
Sep 22 06:28:29 client openvpn[1935]: 2026-09-22 06:28:29 WARNING: Compression for r… set.Sep 22 06:28:29 client systemd[1]: Started OpenVPN Tunneling Application On server.
Hint: Some lines were ellipsized, use -l to show in full.
```

Проверка TAP-туннеля
С client:
```bash
root@client:/home/user# ping -c 4 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.916 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.480 ms
```

Замер скорости в TAP
На server:
```bash
root@server:/home/user# pkill iperf3 2>/dev/null
sleep 1
iperf3 -s &
iperf3: interrupt - the server has terminated
[1]+  Exit 1                  iperf3 -s
[1] 2149
root@server:/home/user# -----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.10.10.2, port 57544
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 57552
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.01   sec  28.9 MBytes   241 Mbits/sec                  
[  5]   1.01-2.01   sec  29.4 MBytes   246 Mbits/sec                  
[  5]   2.01-3.01   sec  30.8 MBytes   258 Mbits/sec                  
[  5]   3.01-4.01   sec  28.2 MBytes   237 Mbits/sec                  
[  5]   4.01-5.01   sec  30.9 MBytes   259 Mbits/sec                  
[  5]   5.01-6.01   sec  31.1 MBytes   261 Mbits/sec                  
[  5]   6.01-7.01   sec  28.4 MBytes   239 Mbits/sec                  
[  5]   7.01-8.01   sec  28.1 MBytes   235 Mbits/sec                  
[  5]   8.01-9.01   sec  28.5 MBytes   239 Mbits/sec                  
[  5]   9.01-10.01  sec  26.2 MBytes   221 Mbits/sec                  
[  5]  10.01-11.01  sec  27.1 MBytes   227 Mbits/sec                  
[  5]  11.01-12.01  sec  28.8 MBytes   241 Mbits/sec                  
[  5]  12.01-13.01  sec  28.0 MBytes   235 Mbits/sec                  
[  5]  13.01-14.01  sec  27.1 MBytes   228 Mbits/sec                  
[  5]  14.01-15.01  sec  27.9 MBytes   234 Mbits/sec                  
[  5]  15.01-16.01  sec  30.6 MBytes   257 Mbits/sec                  
[  5]  16.01-17.01  sec  30.6 MBytes   256 Mbits/sec                  
[  5]  17.01-18.01  sec  26.6 MBytes   223 Mbits/sec                  
[  5]  18.01-19.00  sec  28.0 MBytes   236 Mbits/sec                  
[  5]  19.00-20.01  sec  32.0 MBytes   268 Mbits/sec                  
[  5]  20.01-21.01  sec  31.8 MBytes   266 Mbits/sec                  
[  5]  21.01-22.01  sec  31.5 MBytes   265 Mbits/sec                  
[  5]  22.01-23.01  sec  31.0 MBytes   260 Mbits/sec                  
[  5]  23.01-24.01  sec  31.2 MBytes   262 Mbits/sec                  
[  5]  24.01-25.01  sec  26.2 MBytes   221 Mbits/sec                  
[  5]  25.01-26.00  sec  26.4 MBytes   221 Mbits/sec                  
[  5]  26.00-27.01  sec  31.4 MBytes   263 Mbits/sec                  
[  5]  27.01-28.01  sec  30.5 MBytes   256 Mbits/sec                  
[  5]  28.01-29.01  sec  30.8 MBytes   258 Mbits/sec                  
[  5]  29.01-30.00  sec  31.1 MBytes   262 Mbits/sec                  
[  5]  30.00-31.01  sec  29.9 MBytes   249 Mbits/sec                  
[  5]  31.01-32.01  sec  30.4 MBytes   255 Mbits/sec                  
[  5]  32.01-33.01  sec  27.6 MBytes   232 Mbits/sec                  
[  5]  33.01-34.01  sec  26.4 MBytes   221 Mbits/sec                  
[  5]  34.01-35.01  sec  26.5 MBytes   223 Mbits/sec                  
[  5]  35.01-36.01  sec  26.2 MBytes   220 Mbits/sec                  
[  5]  36.01-37.01  sec  26.6 MBytes   223 Mbits/sec                  
[  5]  37.01-38.01  sec  26.5 MBytes   222 Mbits/sec                  
[  5]  38.01-39.01  sec  26.5 MBytes   223 Mbits/sec                  
[  5]  39.01-40.01  sec  26.5 MBytes   223 Mbits/sec                  
[  5]  40.01-40.04  sec  1.00 MBytes   223 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-40.04  sec  1.13 GBytes   242 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```
На client:
```bash
root@client:/home/user# iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 57552 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec   152 MBytes   254 Mbits/sec  133    215 KBytes       
[  5]   5.00-10.00  sec   142 MBytes   238 Mbits/sec   31    258 KBytes       
[  5]  10.00-15.00  sec   139 MBytes   234 Mbits/sec   92    246 KBytes       
[  5]  15.00-20.00  sec   148 MBytes   248 Mbits/sec   20    255 KBytes       
[  5]  20.00-25.00  sec   152 MBytes   254 Mbits/sec   13    245 KBytes       
[  5]  25.00-30.00  sec   150 MBytes   251 Mbits/sec   36    284 KBytes       
[  5]  30.00-35.00  sec   141 MBytes   236 Mbits/sec   90    218 KBytes       
[  5]  35.00-40.00  sec   132 MBytes   222 Mbits/sec   12    442 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec  1.13 GBytes   242 Mbits/sec  427             sender
[  5]   0.00-40.04  sec  1.13 GBytes   242 Mbits/sec                  receiver
```

# Переключение на TUN на обеих ВМ
```bash 
root@server:/home/user# systemctl restart openvpn@server
sleep 2
ip -br a | grep tun
systemctl status openvpn@server --no-pager
tun0             UNKNOWN        10.10.10.1/24 fe80::bac7:45f7:f928:7963/64 
● openvpn@server.service - OpenVPN Tunneling Application On server
     Loaded: loaded (/etc/systemd/system/openvpn@.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-09-22 06:36:57 UTC; 2s ago
   Main PID: 2198 (openvpn)
     Status: "Pre-connection initialization successful"
      Tasks: 1 (limit: 1008)
     Memory: 1.6M
        CPU: 7ms
     CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
             └─2198 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf

Sep 22 06:36:57 server systemd[1]: openvpn@server.service: Deactivated successfully.
Sep 22 06:36:57 server systemd[1]: Stopped OpenVPN Tunneling Application On server.
Sep 22 06:36:57 server systemd[1]: Starting OpenVPN Tunneling Application On server...
Sep 22 06:36:57 server openvpn[2198]: 2026-09-22 06:36:57 WARNING: Compression for r… set.Sep 22 06:36:57 server systemd[1]: Started OpenVPN Tunneling Application On server.
Hint: Some lines were ellipsized, use -l to show in full.
```

```bash
root@client:/home/user# systemctl stop openvpn@server
sed -i 's/^dev tap$/dev tun/' /etc/openvpn/server.conf
grep '^dev' /etc/openvpn/server.conf
systemctl start openvpn@server
ip -br a | grep tun
dev tun
tun0             UNKNOWN        10.10.10.2/24 fe80::5cce:151c:de37:d03e/64 
```

Проверка TUN
```bash
root@client:/home/user# ping -c 4 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.411 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.293 ms
^C
--- 10.10.10.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1012ms
rtt min/avg/max/mdev = 0.293/0.352/0.411/0.059 ms
```

Замер TUN
```bash
root@server:/home/user# pkill iperf3 2>/dev/null
sleep 1
iperf3 -s &
iperf3: interrupt - the server has terminated
[1]+  Exit 1                  iperf3 -s
[1] 2210
root@server:/home/user# -----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.10.10.2, port 38204
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 38218
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  30.0 MBytes   251 Mbits/sec                  
[  5]   1.00-2.01   sec  30.2 MBytes   253 Mbits/sec                  
[  5]   2.01-3.01   sec  30.9 MBytes   259 Mbits/sec                  
[  5]   3.01-4.01   sec  28.8 MBytes   241 Mbits/sec                  
[  5]   4.01-5.01   sec  28.6 MBytes   240 Mbits/sec                  
[  5]   5.01-6.01   sec  29.0 MBytes   243 Mbits/sec                  
[  5]   6.01-7.01   sec  27.0 MBytes   226 Mbits/sec                  
[  5]   7.01-8.01   sec  30.1 MBytes   253 Mbits/sec                  
[  5]   8.01-9.01   sec  28.6 MBytes   240 Mbits/sec                  
[  5]   9.01-10.01  sec  27.1 MBytes   228 Mbits/sec                  
[  5]  10.01-11.01  sec  26.9 MBytes   225 Mbits/sec                  
[  5]  11.01-12.01  sec  26.9 MBytes   225 Mbits/sec                  
[  5]  12.01-13.01  sec  26.9 MBytes   225 Mbits/sec                  
[  5]  13.01-14.01  sec  26.8 MBytes   224 Mbits/sec                  
[  5]  14.01-15.01  sec  26.9 MBytes   225 Mbits/sec                  
[  5]  15.01-16.01  sec  27.1 MBytes   227 Mbits/sec                  
[  5]  16.01-17.00  sec  27.0 MBytes   228 Mbits/sec                  
[  5]  17.00-18.01  sec  26.9 MBytes   225 Mbits/sec                  
[  5]  18.01-19.01  sec  27.9 MBytes   234 Mbits/sec                  
[  5]  19.01-20.01  sec  28.9 MBytes   243 Mbits/sec                  
[  5]  20.01-21.01  sec  30.5 MBytes   256 Mbits/sec                  
[  5]  21.01-22.01  sec  31.8 MBytes   266 Mbits/sec                  
[  5]  22.01-23.00  sec  28.9 MBytes   243 Mbits/sec                  
[  5]  23.00-24.01  sec  27.6 MBytes   231 Mbits/sec                  
[  5]  24.01-25.01  sec  30.1 MBytes   252 Mbits/sec                  
[  5]  25.01-26.01  sec  30.1 MBytes   253 Mbits/sec                  
[  5]  26.01-27.01  sec  30.6 MBytes   256 Mbits/sec                  
[  5]  27.01-28.00  sec  30.4 MBytes   256 Mbits/sec                  
[  5]  28.00-29.00  sec  30.8 MBytes   258 Mbits/sec                  
[  5]  29.00-30.01  sec  31.1 MBytes   261 Mbits/sec                  
[  5]  30.01-31.01  sec  31.4 MBytes   263 Mbits/sec                  
[  5]  31.01-32.00  sec  31.0 MBytes   261 Mbits/sec                  
[  5]  32.00-33.01  sec  31.0 MBytes   259 Mbits/sec                  
[  5]  33.01-34.01  sec  27.6 MBytes   232 Mbits/sec                  
[  5]  34.01-35.00  sec  26.5 MBytes   222 Mbits/sec                  
[  5]  35.00-36.01  sec  26.5 MBytes   222 Mbits/sec                  
[  5]  36.01-37.01  sec  26.6 MBytes   223 Mbits/sec                  
[  5]  37.01-38.01  sec  27.0 MBytes   226 Mbits/sec                  
[  5]  38.01-39.01  sec  27.1 MBytes   228 Mbits/sec                  
[  5]  39.01-40.01  sec  31.4 MBytes   263 Mbits/sec                  
[  5]  40.01-40.04  sec  1.12 MBytes   264 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-40.04  sec  1.12 GBytes   241 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

```bash
root@client:/home/user# iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 38218 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec   150 MBytes   252 Mbits/sec   47    284 KBytes       
[  5]   5.00-10.00  sec   142 MBytes   238 Mbits/sec   33    301 KBytes       
[  5]  10.00-15.00  sec   135 MBytes   226 Mbits/sec   80    309 KBytes       
[  5]  15.00-20.00  sec   138 MBytes   232 Mbits/sec   35    255 KBytes       
[  5]  20.00-25.00  sec   148 MBytes   249 Mbits/sec   34    307 KBytes       
[  5]  25.00-30.00  sec   153 MBytes   257 Mbits/sec   31    363 KBytes       
[  5]  30.00-35.00  sec   147 MBytes   246 Mbits/sec   31    357 KBytes       
[  5]  35.00-40.00  sec   139 MBytes   234 Mbits/sec   59    211 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec  1.13 GBytes   242 Mbits/sec  350             sender
[  5]   0.00-40.04  sec  1.12 GBytes   241 Mbits/sec                  receiver

iperf Done.
```

Таблица замеров

| Режим | Transfer | Bitrate (receiver) | Retr |
|-------|----------|--------------------|------|
| TAP | 1.13 GBytes | 242 Mbits/sec | 427 |
| TUN | 1.12 GBytes | 241 Mbits/sec | 350 |

## 2. RAS на базе OpenVPN

Отключаем текущий туннель на обеих ВМ
```bash
systemctl stop openvpn@server
systemctl disable openvpn@server
```
Установка Easy-RSA и инициализация PKI
На server:
```bash
apt update
apt install -y easy-rsa
cd /etc/openvpn
/usr/share/easy-rsa/easyrsa init-pki
```

Генерация сертификатов
```bash
cd /etc/openvpn

echo 'rasvpn' | /usr/share/easy-rsa/easyrsa gen-req server nopass
/usr/share/easy-rsa/easyrsa gen-dh

echo 'client' | /usr/share/easy-rsa/easyrsa gen-req client nopass
```
Создадим CA
```bash
root@server:/etc/openvpn# cd /etc/openvpn
/usr/share/easy-rsa/easyrsa build-ca nopass
Using SSL: openssl OpenSSL 3.0.2 15 Mar 2022 (Library: OpenSSL 3.0.2 15 Mar 2022)
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/etc/openvpn/pki/ca.crt
```
Попишем запросы
```bash
cd /etc/openvpn

echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req server server
echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req client client
```

Создаем новый /etc/openvpn/server.conf:
```bash
cat > /etc/openvpn/server.conf <<'EOF'
port 1207
proto udp
dev tun
ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/issued/server.crt
key /etc/openvpn/pki/private/server.key
dh /etc/openvpn/pki/dh.pem
server 10.10.10.0 255.255.255.0
ifconfig-pool-persist ipp.txt
client-to-client
client-config-dir /etc/openvpn/client
keepalive 10 120
comp-lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
EOF
```

Создаем директорию для клиентских конфигов и запускаем сервер
```bash
root@server:/etc/openvpn# mkdir -p /etc/openvpn/client

systemctl daemon-reload
systemctl restart openvpn@server
sleep 2
systemctl status openvpn@server --no-pager
ip -br a | grep tun
ss -lunp | grep 1207
● openvpn@server.service - OpenVPN Tunneling Application On server
     Loaded: loaded (/etc/systemd/system/openvpn@.service; disabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-09-22 06:51:27 UTC; 2s ago
   Main PID: 3259 (openvpn)
     Status: "Initialization Sequence Completed"
      Tasks: 1 (limit: 1008)
     Memory: 3.0M
        CPU: 10ms
     CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
             └─3259 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf

Sep 22 06:51:27 server systemd[1]: Starting OpenVPN Tunneling Application On server...
Sep 22 06:51:27 server openvpn[3259]: 2026-09-22 06:51:27 WARNING: Compression for r… set.Sep 22 06:51:27 server systemd[1]: Started OpenVPN Tunneling Application On server.
Hint: Some lines were ellipsized, use -l to show in full.
tun0             UNKNOWN        10.10.10.1 peer 10.10.10.2/32 fe80::16bc:aa25:a8c5:7937/64 
UNCONN 0      0                 0.0.0.0:1207      0.0.0.0:*    users:(("openvpn",pid=3259,fd=6))    

root@server:/etc/openvpn# systemctl enable openvpn@server
```
Подготовим файлы в /tmp на сервере
```bash
root@server:/etc/openvpn# mkdir -p /tmp/ovpn-client

cp /etc/openvpn/pki/ca.crt             /tmp/ovpn-client/
cp /etc/openvpn/pki/issued/client.crt  /tmp/ovpn-client/
cp /etc/openvpn/pki/private/client.key /tmp/ovpn-client/

chmod 644 /tmp/ovpn-client/ca.crt /tmp/ovpn-client/client.crt
chmod 644 /tmp/ovpn-client/client.key

ls -l /tmp/ovpn-client/
total 16
-rw-r--r-- 1 root root 1204 Sep 22 06:56 ca.crt
-rw-r--r-- 1 root root 4493 Sep 22 06:56 client.crt
-rw-r--r-- 1 root root 1704 Sep 22 06:56 client.key
```
На client скопируем файлы
```bash
root@client:~/openvpn-ras# cd ~/openvpn-ras

scp user@192.168.1.78:/tmp/ovpn-client/ca.crt      ./
scp user@192.168.1.78:/tmp/ovpn-client/client.crt  ./
scp user@192.168.1.78:/tmp/ovpn-client/client.key  ./

chmod 600 client.key
ls -l
user@192.168.1.78's password: 
ca.crt                                                  100% 1204     2.0MB/s   00:00    
user@192.168.1.78's password: 
client.crt                                              100% 4493     6.6MB/s   00:00    
user@192.168.1.78's password: 
client.key                                              100% 1704     2.5MB/s   00:00    
total 16
-rw-r--r-- 1 root root 1204 Sep 22 06:57 ca.crt
-rw-r--r-- 1 root root 4493 Sep 22 06:57 client.crt
-rw------- 1 root root 1704 Sep 22 06:57 client.key
```

На client — создаем client.conf и запускаем
```bash
cat > ~/openvpn-ras/client.conf <<'EOF'
dev tun
proto udp
remote 192.168.1.78 1207
client
resolv-retry infinite
remote-cert-tls server
ca ./ca.crt
cert ./client.crt
key ./client.key
persist-key
persist-tun
comp-lzo
verb 3
EOF

systemctl stop openvpn@server 2>/dev/null
systemctl disable openvpn@server 2>/dev/null

cd ~/openvpn-ras
openvpn --config client.conf

2026-09-22 06:58:09 WARNING: Compression for receiving enabled. Compression has been used in the past to break encryption. Sent packets are not compressed unless "allow-compression yes" is also set.
2026-09-22 06:58:09 --cipher is not set. Previous OpenVPN version defaulted to BF-CBC as fallback when cipher negotiation failed in this case. If you need this fallback please add '--data-ciphers-fallback BF-CBC' to your configuration and/or add BF-CBC to --data-ciphers.
2026-09-22 06:58:09 OpenVPN 2.5.11 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Jul  8 2026
2026-09-22 06:58:09 library versions: OpenSSL 3.0.2 15 Mar 2022, LZO 2.10
2026-09-22 06:58:09 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.1.78:1207
2026-09-22 06:58:09 Socket Buffers: R=[212992->212992] S=[212992->212992]
2026-09-22 06:58:09 UDP link local (bound): [AF_INET][undef]:1194
2026-09-22 06:58:09 UDP link remote: [AF_INET]192.168.1.78:1207
2026-09-22 06:58:09 TLS: Initial packet from [AF_INET]192.168.1.78:1207, sid=a38dc007 d7660b26
2026-09-22 06:58:09 VERIFY OK: depth=1, CN=Easy-RSA CA
2026-09-22 06:58:09 VERIFY KU OK
2026-09-22 06:58:09 Validating certificate extended key usage
2026-09-22 06:58:09 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
2026-09-22 06:58:09 VERIFY EKU OK
2026-09-22 06:58:09 VERIFY OK: depth=0, CN=rasvpn
2026-09-22 06:58:09 Control Channel: TLSv1.3, cipher TLSv1.3 TLS_AES_256_GCM_SHA384, peer certificate: 2048 bit RSA, signature: RSA-SHA256
2026-09-22 06:58:09 [rasvpn] Peer Connection Initiated with [AF_INET]192.168.1.78:1207
2026-09-22 06:58:09 PUSH: Received control message: 'PUSH_REPLY,route 10.10.10.0 255.255.255.0,topology net30,ping 10,ping-restart 120,ifconfig 10.10.10.6 10.10.10.5,peer-id 0,cipher AES-256-GCM'
2026-09-22 06:58:09 OPTIONS IMPORT: timers and/or timeouts modified
2026-09-22 06:58:09 OPTIONS IMPORT: --ifconfig/up options modified
2026-09-22 06:58:09 OPTIONS IMPORT: route options modified
2026-09-22 06:58:09 OPTIONS IMPORT: peer-id set
2026-09-22 06:58:09 OPTIONS IMPORT: adjusting link_mtu to 1625
2026-09-22 06:58:09 OPTIONS IMPORT: data channel crypto options modified
2026-09-22 06:58:09 Data Channel: using negotiated cipher 'AES-256-GCM'
2026-09-22 06:58:09 Outgoing Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
2026-09-22 06:58:09 Incoming Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
2026-09-22 06:58:09 net_route_v4_best_gw query: dst 0.0.0.0
2026-09-22 06:58:09 net_route_v4_best_gw result: via 192.168.1.1 dev ens18
2026-09-22 06:58:09 ROUTE_GATEWAY 192.168.1.1/255.255.255.0 IFACE=ens18 HWADDR=bc:24:11:08:01:7f
2026-09-22 06:58:09 TUN/TAP device tun0 opened
2026-09-22 06:58:09 net_iface_mtu_set: mtu 1500 for tun0
2026-09-22 06:58:09 net_iface_up: set tun0 up
2026-09-22 06:58:09 net_addr_ptp_v4_add: 10.10.10.6 peer 10.10.10.5 dev tun0
2026-09-22 06:58:09 net_route_v4_add: 10.10.10.0/24 via 10.10.10.5 dev [NULL] table 0 metric -1
2026-09-22 06:58:09 Initialization Sequence Completed
```
Проверка на клиенте
```bash
root@client:/home/user# ip -br a | grep tun
ping -c 4 10.10.10.1
ip r
tun0             UNKNOWN        10.10.10.6 peer 10.10.10.5/32 fe80::cea9:4ea2:9d39:78/64 
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.371 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.316 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.205 ms
^C
--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2045ms
rtt min/avg/max/mdev = 0.205/0.297/0.371/0.069 ms
default via 192.168.1.1 dev ens18 proto dhcp src 192.168.1.104 metric 100 
10.10.10.0/24 via 10.10.10.5 dev tun0 
10.10.10.5 dev tun0 proto kernel scope link src 10.10.10.6 
192.168.1.0/24 dev ens18 proto kernel scope link src 192.168.1.104 metric 100 
192.168.1.1 dev ens18 proto dhcp scope link src 192.168.1.104 metric 100 
```

Проверка на сервере
```bash
root@server:/etc/openvpn# cat /var/log/openvpn-status.log
ip -br a | grep tun
OpenVPN CLIENT LIST
Updated,2026-09-22 07:46:05
Common Name,Real Address,Bytes Received,Bytes Sent,Connected Since
client,192.168.1.104:1194,18577,18253,2026-09-22 06:58:09
ROUTING TABLE
Virtual Address,Common Name,Real Address,Last Ref
10.10.10.6,client,192.168.1.104:1194,2026-09-22 07:00:47
GLOBAL STATS
Max bcast/mcast queue length,1
END
tun0             UNKNOWN        10.10.10.1 peer 10.10.10.2/32 fe80::16bc:aa25:a8c5:7937/64
```

