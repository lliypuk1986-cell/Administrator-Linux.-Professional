# Домашнее задание "Строим бонды и вланы"

## Цель
Научиться настраивать VLAN и LACP.

## Задание
в Office1 в тестовой подсети появляется сервера с доп интерфейсами и адресами
в internal сети testLAN:
testClient1 - 10.10.10.254
testClient2 - 10.10.10.254
testServer1- 10.10.10.1
testServer2- 10.10.10.1
Равести вланами:
testClient1 <-> testServer1
testClient2 <-> testServer2

Между centralRouter и inetRouter "пробросить" 2 линка (общая inernal сеть) и объединить их в бонд, проверить работу c отключением интерфейсов

---

## Выполнение задания
Для выполнения задания будет использоваться ОС Ubuntu 22.04.5 LTS

Создаем на proxmox 7 виртуальных машин.
inetRouter: IP 192.168.1.116/24
centralRouter: IP 192.168.1.72/24
office1Router: IP 192.168.1.132/24
testClient1: IP 192.168.1.86/24
testServer1: IP 192.168.1.115/24
testClient2: IP 192.168.1.122/24
testServer2: IP 192.168.1.71/24

Нам нужно 4 bridge:
vmbr1	testLAN (VLAN 1 и VLAN 2)
vmbr10	Первый линк bond между inetRouter и centralRouter
vmbr20	Второй линк bond между inetRouter и centralRouter
vmbr30	Линк centralRouter ↔ office1Router

![alt text](image.png)

## Карта подключений

| VM | net0 | net1 | net2 | net3 |
|---|---|---|---|---|
| inetRouter | vmbr0 (mgmt) | vmbr10 | vmbr20 | — |
| centralRouter | vmbr0 (mgmt) | vmbr10 | vmbr20 | vmbr30 |
| office1Router | vmbr0 (mgmt) | vmbr30 | vmbr1 | — |
| testClient1 | vmbr0 (mgmt) | vmbr1 | — | — |
| testServer1 | vmbr0 (mgmt) | vmbr1 | — | — |
| testClient2 | vmbr0 (mgmt) | vmbr1 | — | — |
| testServer2 | vmbr0 (mgmt) | vmbr1 | — | — |

Пропишем на каждой ВМ /etc/hosts:

```bash
sudo nano /etc/hosts

127.0.0.1       localhost
127.0.1.1       testClient1 (подставляем текущую ВМ)

192.168.1.116   inetRouter
192.168.1.72    centralRouter
192.168.1.132   office1Router
192.168.1.86    testClient1
192.168.1.115   testServer1
192.168.1.122   testClient2
192.168.1.71    testServer2
```
Проверяем
```bash
ping -c 2 inetRouter
ping -c 2 testServer1

root@inetRouter:/home/user# ping -c 2 inetRouter
ping -c 2 testServer1
PING inetRouter (127.0.1.1) 56(84) bytes of data.
64 bytes from inetRouter (127.0.1.1): icmp_seq=1 ttl=64 time=0.037 ms
64 bytes from inetRouter (127.0.1.1): icmp_seq=2 ttl=64 time=0.021 ms

--- inetRouter ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1028ms
rtt min/avg/max/mdev = 0.021/0.029/0.037/0.008 ms
PING testServer1 (192.168.1.115) 56(84) bytes of data.
64 bytes from testServer1 (192.168.1.115): icmp_seq=1 ttl=64 time=0.506 ms
64 bytes from testServer1 (192.168.1.115): icmp_seq=2 ttl=64 time=0.297 ms

--- testServer1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.297/0.401/0.506/0.104 ms

root@centralRouter:/home/user# ping -c 2 inetRouter
ping -c 2 testServer1
PING inetRouter (192.168.1.116) 56(84) bytes of data.
64 bytes from inetRouter (192.168.1.116): icmp_seq=1 ttl=64 time=0.659 ms
64 bytes from inetRouter (192.168.1.116): icmp_seq=2 ttl=64 time=0.287 ms

--- inetRouter ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1029ms
rtt min/avg/max/mdev = 0.287/0.473/0.659/0.186 ms
PING testServer1 (192.168.1.115) 56(84) bytes of data.
64 bytes from testServer1 (192.168.1.115): icmp_seq=1 ttl=64 time=0.365 ms
64 bytes from testServer1 (192.168.1.115): icmp_seq=2 ttl=64 time=0.294 ms

--- testServer1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.294/0.329/0.365/0.035 ms

root@office1Router:/home/user# ping -c 2 inetRouter
ping -c 2 testServer1
PING inetRouter (192.168.1.116) 56(84) bytes of data.
64 bytes from inetRouter (192.168.1.116): icmp_seq=1 ttl=64 time=0.552 ms
64 bytes from inetRouter (192.168.1.116): icmp_seq=2 ttl=64 time=0.311 ms

--- inetRouter ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.311/0.431/0.552/0.120 ms
PING testServer1 (192.168.1.115) 56(84) bytes of data.
64 bytes from testServer1 (192.168.1.115): icmp_seq=1 ttl=64 time=0.526 ms
64 bytes from testServer1 (192.168.1.115): icmp_seq=2 ttl=64 time=0.205 ms

--- testServer1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.205/0.365/0.526/0.160 ms

root@testClient1:/home/user# ping -c 2 inetRouter
ping -c 2 testServer1
PING inetRouter (192.168.1.116) 56(84) bytes of data.
64 bytes from inetRouter (192.168.1.116): icmp_seq=1 ttl=64 time=0.611 ms
64 bytes from inetRouter (192.168.1.116): icmp_seq=2 ttl=64 time=0.291 ms

--- inetRouter ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1013ms
rtt min/avg/max/mdev = 0.291/0.451/0.611/0.160 ms
PING testServer1 (192.168.1.115) 56(84) bytes of data.
64 bytes from testServer1 (192.168.1.115): icmp_seq=1 ttl=64 time=0.293 ms
64 bytes from testServer1 (192.168.1.115): icmp_seq=2 ttl=64 time=0.217 ms

--- testServer1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.217/0.255/0.293/0.038 ms
root@testClient1:/home/user# 

root@testServer1:/home/user# ping -c 2 inetRouter
ping -c 2 testServer1
PING inetRouter (192.168.1.116) 56(84) bytes of data.
64 bytes from inetRouter (192.168.1.116): icmp_seq=1 ttl=64 time=0.188 ms
64 bytes from inetRouter (192.168.1.116): icmp_seq=2 ttl=64 time=0.271 ms

--- inetRouter ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1004ms
rtt min/avg/max/mdev = 0.188/0.229/0.271/0.041 ms
PING testServer1 (127.0.1.1) 56(84) bytes of data.
64 bytes from testServer1 (127.0.1.1): icmp_seq=1 ttl=64 time=0.015 ms
64 bytes from testServer1 (127.0.1.1): icmp_seq=2 ttl=64 time=0.024 ms

--- testServer1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.015/0.019/0.024/0.004 ms

root@testClient2:/home/user# ping -c 2 inetRouter
ping -c 2 testServer1
PING inetRouter (192.168.1.116) 56(84) bytes of data.
64 bytes from inetRouter (192.168.1.116): icmp_seq=1 ttl=64 time=0.592 ms
64 bytes from inetRouter (192.168.1.116): icmp_seq=2 ttl=64 time=0.263 ms

--- inetRouter ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1008ms
rtt min/avg/max/mdev = 0.263/0.427/0.592/0.164 ms
PING testServer1 (192.168.1.115) 56(84) bytes of data.
64 bytes from testServer1 (192.168.1.115): icmp_seq=1 ttl=64 time=0.488 ms
64 bytes from testServer1 (192.168.1.115): icmp_seq=2 ttl=64 time=0.238 ms

--- testServer1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.238/0.363/0.488/0.125 ms

root@testServer2:/home/user# ping -c 2 inetRouter
ping -c 2 testServer1
PING inetRouter (192.168.1.116) 56(84) bytes of data.
64 bytes from inetRouter (192.168.1.116): icmp_seq=1 ttl=64 time=0.535 ms
64 bytes from inetRouter (192.168.1.116): icmp_seq=2 ttl=64 time=0.260 ms

--- inetRouter ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1026ms
rtt min/avg/max/mdev = 0.260/0.397/0.535/0.137 ms
PING testServer1 (192.168.1.115) 56(84) bytes of data.64 bytes from testServer1 (192.168.1.115): icmp_seq=1 ttl=64 time=0.503 ms
64 bytes from testServer1 (192.168.1.115): icmp_seq=2 ttl=64 time=0.260 ms

--- testServer1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.260/0.381/0.503/0.121 ms
root@testServer2:/home/user# 
```

Настроим VLAN 1 на testClient1
```bash
root@testClient1:/home/user# sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      addresses: [10.10.10.254/24]


root@testClient1:/home/user# sudo netplan generate

root@testClient1:/home/user# sudo netplan try
Do you want to keep these settings?
Press ENTER before the timeout to accept the new configuration
Changes will revert in 120 secoChanges will revert in 119 secoChanges will revert in 118 secoChanges w

root@testClient1:/home/user# sudo netplan apply

root@testClient1:/home/user# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.1.86/24 metric 100 fe80::be24:11ff:fe24:e9c5/64 
ens19            UP             10.10.10.254/24 fe80::be24:11ff:fe40:c859/64 
```
Настроим  VLAN 1 на testServer1
```bash
root@testServer1:/home/user# sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      addresses: [10.10.10.1/24]


root@testServer1:/home/user# sudo netplan generate

root@testServer1:/home/user# sudo netplan try
Do you want to keep these settings?
Press ENTER before the timeout to accept the new configuration

Changes will revert in 120 secoChanges will revert in 119 secoChanges will revert in 118 seconds
Configuration accepted.

root@testServer1:/home/user# sudo netplan apply

root@testServer1:/home/user# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.1.115/24 metric 100 fe80::be24:11ff:feed:b402/64 
ens19            UP             10.10.10.1/24 fe80::be24:11ff:fed6:9468/64 
```

Проверка VLAN 1
```bash
root@testClient1:/home/user# ping -c 3 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.345 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.266 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.215 ms

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2053msrtt min/avg/max/mdev = 0.215/0.275/0.345/0.053 ms

root@testServer1:/home/user# ping -c 3 10.10.10.254
PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=0.221 ms
64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=0.232 ms
64 bytes from 10.10.10.254: icmp_seq=3 ttl=64 time=0.279 ms

--- 10.10.10.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2050ms
rtt min/avg/max/mdev = 0.221/0.244/0.279/0.025 ms


```

Настроим netplan на testClient2

```bash
sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      dhcp4: no
      optional: true
      addresses: [10.10.10.254/24]

sudo netplan generate
sudo netplan apply
user# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.1.122/24 metric 100 fe80::be24:11ff:fecc:89a1/64 
ens19            UP             10.10.10.254/24 fe80::be24:11ff:fe4e:8c26/64 
```
Настроим netplan на testServer2

```bash
sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      dhcp4: no
      optional: true
      addresses: [10.10.10.1/24]

sudo netplan apply
root@testServer2:/home/user# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.1.71/24 metric 100 fe80::be24:11ff:fe2a:604e/64 
ens19            UP             10.10.10.1/24 fe80::be24:11ff:feb2:f317/64 
```

Проверка VLAN 2
```bash
root@testClient2:/home/user# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.1.122/24 metric 100 fe80::be24:11ff:fecc:89a1/64 
ens19            UP             10.10.10.254/24 fe80::be24:11ff:fe4e:8c26/64 

root@testClient2:/home/user# ping -c 3 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.439 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.215 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.248 ms

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2030ms
rtt min/avg/max/mdev = 0.215/0.300/0.439/0.098 ms


root@testServer2:/home/user# ping -c 3 10.10.10.254
PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=0.279 ms
64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=0.250 ms
64 bytes from 10.10.10.254: icmp_seq=3 ttl=64 time=0.290 ms

--- 10.10.10.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2030msrtt min/avg/max/mdev = 0.250/0.273/0.290/0.016 ms
```

Проверка изоляции VLAN 1 ↔ VLAN 2
```bash
root@testClient1:/home/user# ping -c 3 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.215 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.290 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.291 ms

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2036msrtt min/avg/max/mdev = 0.215/0.265/0.291/0.035 ms

root@testServer2:/home/user# sudo tcpdump -i ens19 -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens19, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel

```
Все три проверки прошли:
VLAN 1: testClient1 ↔ testServer1 — ping OK ✅
VLAN 2: testClient2 ↔ testServer2 — ping OK ✅
Изоляция: на testServer2 при ping с testClient1 в VLAN 1 поймано 0 пакетов ✅


Настроим bond0 на inetRouter
```bash
sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      dhcp4: no
      optional: true
    ens20:
      dhcp4: no
      optional: true
  bonds:
    bond0:
      interfaces: [ens19, ens20]
      addresses: [192.168.255.1/30]
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4

root@inetRouter:/home/user# sudo netplan generate
root@inetRouter:/home/user# sudo netplan apply

root@inetRouter:/home/user# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.1.116/24 metric 100 fe80::be24:11ff:fe63:1a9a/64 
ens19            UP             
ens20            UP             
bond0            UP             192.168.255.1/30 fe80::b0d2:86ff:fe13:49a/64 

root@inetRouter:/home/user# cat /proc/net/bonding/bond0
Ethernet Channel Bonding Driver: v5.15.0-186-generic

Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer3+4 (1)
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0
Peer Notification Delay (ms): 0
802.3ad info
LACP active: on
LACP rate: fast
Min links: 0
Aggregator selection policy (ad_select): stable
System priority: 65535
System MAC address: b2:d2:86:13:04:9a
Active Aggregator Info:
        Aggregator ID: 1
        Number of ports: 1
        Actor Key: 0
        Partner Key: 1
        Partner Mac Address: 00:00:00:00:00:00

Slave Interface: ens20
MII Status: up
Speed: Unknown
Duplex: Unknown
Link Failure Count: 0
Permanent HW addr: bc:24:11:f8:27:eb
Slave queue ID: 0
Aggregator ID: 1
Actor Churn State: monitoring
Partner Churn State: monitoringActor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: b2:d2:86:13:04:9a
    port key: 0
    port priority: 255
    port number: 1
    port state: 79
details partner lacp pdu:
    system priority: 65535
    system mac address: 00:00:00:00:00:00
    oper key: 1
    port priority: 255
    port number: 1
    port state: 1

Slave Interface: ens19
MII Status: up
Speed: Unknown
Duplex: Unknown
Link Failure Count: 0
Permanent HW addr: bc:24:11:fb:8d:f9
Slave queue ID: 0
Aggregator ID: 2
Actor Churn State: monitoring
Partner Churn State: monitoringActor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: b2:d2:86:13:04:9a
    port key: 0
    port priority: 255
    port number: 2
    port state: 71
details partner lacp pdu:
    system priority: 65535
    system mac address: 00:00:00:00:00:00
    oper key: 1
    port priority: 255
    port number: 1
    port state: 1

```


Настроим bond0 и линк до office1Router на centralRouter
```bash
sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      dhcp4: no
      optional: true
    ens20:
      dhcp4: no
      optional: true
    ens21:
      dhcp4: no
      optional: true
      addresses: [192.168.255.9/30]
  bonds:
    bond0:
      interfaces: [ens19, ens20]
      addresses: [192.168.255.2/30]
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4

root@centralRouter:/home/user# sudo netplan generate
root@centralRouter:/home/user# sudo netplan apply

root@centralRouter:/home/user# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.1.72/24 metric 100 fe80::be24:11ff:fe8c:c42f/64 
ens19            UP             
ens20            UP             
ens21            UP             192.168.255.9/30 fe80::be24:11ff:fe0d:6481/64 
bond0            UP             192.168.255.2/30 fe80::b0d2:86ff:fe13:49a/64 

root@centralRouter:/home/user# cat /proc/net/bonding/bond0
Ethernet Channel Bonding Driver: v5.15.0-186-generic

Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer3+4 (1)
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0
Peer Notification Delay (ms): 0
802.3ad info
LACP active: on
LACP rate: fast
Min links: 0
Aggregator selection policy (ad_select): stable
System priority: 65535
System MAC address: b2:d2:86:13:04:9a
Active Aggregator Info:
        Aggregator ID: 1
        Number of ports: 1
        Actor Key: 0
        Partner Key: 1
        Partner Mac Address: 00:00:00:00:00:00

Slave Interface: ens19
MII Status: up
Speed: Unknown
Duplex: Unknown
Link Failure Count: 0
Permanent HW addr: bc:24:11:6b:2f:46
Slave queue ID: 0
Aggregator ID: 1
Actor Churn State: monitoring
Partner Churn State: monitoringActor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: b2:d2:86:13:04:9a
    port key: 0
    port priority: 255
    port number: 1
    port state: 79
details partner lacp pdu:
    system priority: 65535
    system mac address: 00:00:00:00:00:00
    oper key: 1
    port priority: 255
    port number: 1
    port state: 1

Slave Interface: ens20
MII Status: up
Speed: Unknown
Duplex: Unknown
Link Failure Count: 0
Permanent HW addr: bc:24:11:33:a4:e6
Slave queue ID: 0
Aggregator ID: 2
Actor Churn State: monitoring
Partner Churn State: monitoringActor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: b2:d2:86:13:04:9a
    port key: 0
    port priority: 255
    port number: 2
    port state: 71
details partner lacp pdu:
    system priority: 65535
    system mac address: 00:00:00:00:00:00
    oper key: 1
    port priority: 255
    port number: 1
    port state: 1
```


Настроим линк на office1Router
```bash
sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      dhcp4: no
      optional: true
      addresses: [192.168.255.10/30]
    ens20:
      dhcp4: no
      optional: true

root@office1Router:/home/user# sudo netplan generate
root@office1Router:/home/user# sudo netplan apply

root@office1Router:/home/user# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.1.132/24 metric 100 fe80::be24:11ff:fe3c:771b/64 
ens19            UP             192.168.255.10/30 fe80::be24:11ff:fe5a:49f3/64 
ens20            UP             fe80::be24:11ff:fe2c:157d/64
```

Проверим связность по всем линкам

```bash
root@inetRouter:/home/user# ping -c 3 192.168.255.2
PING 192.168.255.2 (192.168.255.2) 56(84) bytes of data.
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.514 ms
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.237 ms
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.197 ms


root@centralRouter:/home/user# ping -c 3 192.168.255.1
ping -c 3 192.168.255.10
PING 192.168.255.1 (192.168.255.1) 56(84) bytes of data.
64 bytes from 192.168.255.1: icmp_seq=1 ttl=64 time=0.522 ms
64 bytes from 192.168.255.1: icmp_seq=1 ttl=64 time=0.247 ms
64 bytes from 192.168.255.1: icmp_seq=1 ttl=64 time=0.214 ms


PING 192.168.255.10 (192.168.255.10) 56(84) bytes of data.
64 bytes from 192.168.255.10: icmp_seq=1 ttl=64 time=0.526 ms
64 bytes from 192.168.255.10: icmp_seq=2 ttl=64 time=0.235 ms
64 bytes from 192.168.255.10: icmp_seq=3 ttl=64 time=0.273 ms


```

Тест отказоустойчивости
```bash
root@inetRouter:/home/user# ping 192.168.255.2
PING 192.168.255.2 (192.168.255.2) 56(84) bytes of data.
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.126 ms
64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=0.234 ms
64 bytes from 192.168.255.2: icmp_seq=3 ttl=64 time=0.513 ms

root@centralRouter:/home/user# sudo ip link set ens19 down

root@inetRouter:/home/user# ping 192.168.255.2
PING 192.168.255.2 (192.168.255.2) 56(84) bytes of data.
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.235 ms
64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=0.421 ms
From 192.168.255.1 icmp_seq=3 Destination Host Unreachable
From 192.168.255.1 icmp_seq=4 Destination Host Unreachable
From 192.168.255.1 icmp_seq=5 Destination Host Unreachable
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.234 ms
64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=0.124 ms


root@centralRouter:/home/user# sudo ip link set ens19 up


root@inetRouter:/home/user# ping 192.168.255.2
PING 192.168.255.2 (192.168.255.2) 56(84) bytes of data.
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.431 ms
64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=0.126 ms
64 bytes from 192.168.255.2: icmp_seq=3 ttl=64 time=0.224 ms

root@centralRouter:/home/user# sudo ip link set ens20 down

root@inetRouter:/home/user# ping 192.168.255.2
PING 192.168.255.2 (192.168.255.2) 56(84) bytes of data.
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.124 ms
64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=0.435 ms
From 192.168.255.1 icmp_seq=3 Destination Host Unreachable
From 192.168.255.1 icmp_seq=4 Destination Host Unreachable
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=0.261 ms
64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=0.324 ms
```
Тест отказоустойчивости прошёл — при отключении ens19 и ens20 ping не прерывался полностью: было 2–3 потерянных пакета на переключение, затем трафик восстанавливался и шёл через оставшийся slave. То же самое при отключении ens20. Значит, bond выполняет функцию резервирования.