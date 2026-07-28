# Домашнее задание "Резервное копирование"

## Цель
использовать политики и методики резерного копирования;
работать с инструментами rsync, tar, dd и bacula.

## Задание
Настроить удаленный бэкап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:
- директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB; (Студент самостоятельно настраивает)
- репозиторий для резервных копий должен быть зашифрован ключом или паролем - на усмотрение студента;
- имя бэкапа должно содержать информацию о времени снятия бекапа;
- глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;
- резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
- написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на усмотрение студента;
- настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.

---

## Выполнение задания

Для выполнения задания будет использоваться ОС Ubuntu 22.04.5 LTS
Создаем ВМ:
backup: IP 192.168.1.44/24
client: IP 192.168.1.56/24

# 1. Установка BorgBackup
На обеих машинах (client и backup) выполняем:
```bash
root@backup:~# sudo apt update && sudo apt install -y borgbackup
root@client:~#  sudo apt update && sudo apt install -y borgbackup
```
# 2. Создаём пользователя borg и каталог для резервных копий
```bash
root@backup:~# sudo useradd -m -s /bin/bash borg
root@backup:~# sudo passwd borg
New password: 12345
Retype new password: 12345
passwd: password updated successfully
root@backup:~# sudo mkdir -p /var/backup
root@backup:~# sudo chown -R borg:borg /var/backup
```
Настраиваем SSH для пользователя borg
Переключаемся на borg и готовим authorized_keys:
```bash
root@backup:~# sudo su - borg
borg@backup:~$ mkdir -p ~/.ssh
borg@backup:~$ touch ~/.ssh/authorized_keys
borg@backup:~$ chmod 700 ~/.ssh
borg@backup:~$ chmod 600 ~/.ssh/authorized_keys
```

# 3. Настройка клиента
3.1. Генерируем SSH-ключ и копируем его на backup
На клиенте выполняем:
```bash
user@client:~$ ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
Generating public/private ed25519 key pair.
Your identification has been saved in /home/user/.ssh/id_ed25519
Your public key has been saved in /home/user/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:V9WuOyublsCBzOKlJL1KfnmT3OFACdtIAOQpqPIT4oo user@client
The key's randomart image is:
+--[ED25519 256]--+
| .o...        .. |
|.. .  o      .  .|
|o o  o B o  .  . |
|..  . * O ..    .|
|+ .  + *S...   . |
|oo .. + ..+   .  |
| .oo . o = o . . |
|o  .o o = o +.o  |
|E    . . . .oo.o |
+----[SHA256]-----+
user@client:~$ ssh-copy-id -i ~/.ssh/id_ed25519.pub borg@192.168.1.44
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/user/.ssh/id_ed25519.pub"
The authenticity of host '192.168.1.44 (192.168.1.44)' can't be established.
ED25519 key fingerprint is SHA256:SkurvVt/IoXngSPq02Stf1NgOgfkDB9+65YOuK9zOsg.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
borg@192.168.1.44's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'borg@192.168.1.44'"
and check to make sure that only the key(s) you wanted were added.
```
Проверяем беспарольный доступ:

```bash
user@client:~$ ssh borg@192.168.1.44 hostname
backup
```
3.2. Инициализируем репозиторий Borg
Задаём надёжную парольную фразу для шифрования. Выполняем на клиенте:
```bash
user@client:~$ borg init --encryption=repokey borg@192.168.1.44:/var/backup/
Enter new passphrase: 
Enter same passphrase again: 
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "12345"
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.1.44/var/backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
If you used a repokey mode, the key is stored in the repo, but you should back it up separately.
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).
```
3.3. Ручной тест создания бэкапа
```bash
borg create --stats --list \
    borg@192.168.1.44:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc

------------------------------------------------------------------------------
Repository: ssh://borg@192.168.1.44/var/backup
Archive name: etc-2026-07-28_14:56:10
Archive fingerprint: a5f225e6cf3b0d8087f94f3188f6aa20c98dc40d75d28ab9db80a58d73da9e7d
Time (start): Tue, 2026-07-28 14:56:23
Time (end):   Tue, 2026-07-28 14:56:25
Duration: 1.61 seconds
Number of files: 704
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:                2.15 MB            958.57 kB            913.02 kB
All archives:                2.15 MB            957.96 kB            981.42 kB

                       Unique chunks         Total chunks
Chunk index:                     662                  695
------------------------------------------------------------------------------
```
Смотрим список архивов, их содержимое и восстанавливаем файл:
```bash
user@client:~$ borg list borg@192.168.1.44:/var/backup/
Enter passphrase for key ssh://borg@192.168.1.44/var/backup: 
etc-2026-07-28_14:56:10              Tue, 2026-07-28 14:56:23 [a5f225e6cf3b0d8087f94f3188f6aa20c98dc40d75d28ab9db80a58d73da9e7d]

user@client:~$ borg list borg@192.168.1.44:/var/backup/::etc-2026-07-28_14:56:10
user@client:~$ borg extract borg@192.168.1.44:/var/backup/::etc-2026-07-28_14:56:10 etc/hostname
```

# 4. Автоматизация резервного копирования
4.1. Пишем скрипт резервного копирования
Создаём на client файл /usr/local/bin/borg-backup.sh:
```bash
sudo nano /usr/local/bin/borg-backup.sh
```
Содержимое:
```bash
#!/bin/bash

export BORG_REPO="borg@192.168.1.44:/var/backup/"
export BORG_PASSPHRASE="Ваш_пароль_от_репозитория"   # заменяем на реальный пароль
BACKUP_TARGET="/etc"
LOG_TAG="borg-backup"

echo "=== Start backup: $(date) ===" | logger -t "$LOG_TAG"

# Создание бэкапа
borg create --stats --list \
    ::etc-{now:%Y-%m-%d_%H:%M:%S} \
    $BACKUP_TARGET \
    2>&1 | logger -t "$LOG_TAG"

if [ ${PIPESTATUS[0]} -ne 0 ]; then
    echo "ERROR: borg create failed" | logger -t "$LOG_TAG"
    exit 1
fi

# Проверка целостности репозитория
borg check 2>&1 | logger -t "$LOG_TAG"

# Очистка старых копий согласно политике хранения
borg prune --list \
    --keep-daily   90 \
    --keep-monthly 12 \
    --keep-yearly   1 \
    2>&1 | logger -t "$LOG_TAG"

echo "=== Backup finished: $(date) ===" | logger -t "$LOG_TAG"
```

Делаем скрипт исполняемым:
```bash
root@client:~# sudo chmod +x /usr/local/bin/borg-backup.sh
```

4.2. Создаём systemd-сервис и таймер
Сервис /etc/systemd/system/borg-backup.service:

```bash
cat > /etc/systemd/system/borg-backup.service << 'EOF'
[Unit]
Description=Borg Backup Service
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/borg-backup.sh
StandardOutput=journal
StandardError=journal
SyslogIdentifier=borg-backup
EOF
```

Таймер /etc/systemd/system/borg-backup.timer:
```bash
sudo nano /etc/systemd/system/borg-backup.timer

ini
[Unit]
Description=Borg Backup Timer

[Timer]
OnUnitActiveSec=5min
Persistent=true

[Install]
WantedBy=timers.target
```

4.3. Запускаем и проверяем
```bash
root@client:~# sudo systemctl daemon-reload
root@client:~# sudo systemctl enable --now borg-backup.timer
Created symlink /etc/systemd/system/timers.target.wants/borg-backup.timer → /etc/systemd/system/borg-backup.timer.
```

Проверяем, что таймер активен:

```bash
root@client:~# systemctl list-timers borg-backup.timer
NEXT LEFT LAST PASSED UNIT              ACTIVATES          
n/a  n/a  n/a  n/a    borg-backup.timer borg-backup.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.
```

# 5. Исправление: настройка SSH-ключа для root
Генерируем ключ для root (на client)
```bash
root@client:~# ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519
Generating public/private ed25519 key pair.
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:IfNi05K9DJa5nU1I9L8fzTK4skRdsPoSvwSch9n6MoE root@client
The key's randomart image is:
+--[ED25519 256]--+
|        .   .    |
|       . .   o   |
|      o o . . .  |
|       @ + O .   |
|      @ S.@ =    |
|     o OE*.B o o |
|      . = =.* + o|
|         .++ + + |
|          .=+ .  |
+----[SHA256]-----+
```
Копируем публичный ключ на backup
При запросе вводим пароль пользователя borg:
```bash
root@client:~# ssh-copy-id -i /root/.ssh/id_ed25519.pub borg@192.168.1.44
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
borg@192.168.1.44's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'borg@192.168.1.44'"
and check to make sure that only the key(s) you wanted were added.

```
Проверяем беспарольный доступ от root
```bash
root@client:~# ssh borg@192.168.1.44 hostname
backup
root@client:~# ssh -o StrictHostKeyChecking=accept-new borg@192.168.1.44 hostname
backup
```
Запускаем сервис заново
```bash
root@client:~# systemctl start borg-backup.service
root@client:~# systemctl status borg-backup.service
○ borg-backup.service - Borg Backup Service
     Loaded: loaded (/etc/systemd/system/borg-backup.service; static)
     Active: inactive (dead) since Tue 2026-07-28 18:44:49 UTC; 10ms ago
TriggeredBy: ● borg-backup.timer
    Process: 6878 ExecStart=/usr/local/bin/borg-backup.sh (code=exited, status=0/SUCCESS)
   Main PID: 6878 (code=exited, status=0/SUCCESS)
        CPU: 1.415s

Jul 28 18:44:46 client borg-backup[6883]: Chunk index:                     696                 2861
Jul 28 18:44:46 client borg-backup[6883]: ------------------------------------------------------------------------------
Jul 28 18:44:48 client borg-backup[6889]: Keeping archive (rule: daily #1):        etc-2026-07-28_18:44:44              >Jul 28 18:44:48 client borg-backup[6889]: Pruning archive (1/2):                   etc-2026-07-28_18:37:51              >Jul 28 18:44:48 client borg-backup[6889]: Pruning archive (2/2):                   etc-2026-07-28_18:37:24              >Jul 28 18:44:48 client borg-backup[6889]: Keeping archive (rule: daily[oldest] #2): etc-2026-07-28_14:56:10             >Jul 28 18:44:49 client borg-backup[6892]: === Backup finished: Tue Jul 28 06:44:49 PM UTC 2026 ===
Jul 28 18:44:49 client systemd[1]: borg-backup.service: Deactivated successfully.
Jul 28 18:44:49 client systemd[1]: Finished Borg Backup Service.
Jul 28 18:44:49 client systemd[1]: borg-backup.service: Consumed 1.415s CPU time.
```

Смотрим логи в реальном времени:

```bash
root@client:~# journalctl -u borg-backup.service -f
Jul 28 18:44:46 client borg-backup[6883]: Chunk index:                     696                 2861
Jul 28 18:44:46 client borg-backup[6883]: ------------------------------------------------------------------------------
Jul 28 18:44:48 client borg-backup[6889]: Keeping archive (rule: daily #1):        etc-2026-07-28_18:44:44              Tue, 2026-07-28 18:44:45 [bd5865f284b01a78ab5f33037b39f774922340210bdb758409174fa494ac264d]
Jul 28 18:44:48 client borg-backup[6889]: Pruning archive (1/2):                   etc-2026-07-28_18:37:51              Tue, 2026-07-28 18:37:54 [9785a9a9da890c8fd38897d7dd18cd554c3fe85b6b8d849c1e87db101ab398af]
Jul 28 18:44:48 client borg-backup[6889]: Pruning archive (2/2):                   etc-2026-07-28_18:37:24              Tue, 2026-07-28 18:37:32 [8062ba19d85b13565c3fd1138d96987fe7329cb85c0801e1353a73eaaff76209]
Jul 28 18:44:48 client borg-backup[6889]: Keeping archive (rule: daily[oldest] #2): etc-2026-07-28_14:56:10              Tue, 2026-07-28 14:56:23 [a5f225e6cf3b0d8087f94f3188f6aa20c98dc40d75d28ab9db80a58d73da9e7d]
Jul 28 18:44:49 client borg-backup[6892]: === Backup finished: Tue Jul 28 06:44:49 PM UTC 2026 ===
Jul 28 18:44:49 client systemd[1]: borg-backup.service: Deactivated successfully.
Jul 28 18:44:49 client systemd[1]: Finished Borg Backup Service.
Jul 28 18:44:49 client systemd[1]: borg-backup.service: Consumed 1.415s CPU time.
```
или через syslog:
```bash
root@client:~# tail -f /var/log/syslog | grep borg-backup
Jul 28 18:49:54 client borg-backup: U /etc/passwd-
Jul 28 18:49:54 client borg-backup: U /etc/hosts.allow
Jul 28 18:49:54 client borg-backup: U /etc/hosts.deny
Jul 28 18:49:54 client borg-backup: U /etc/passwd
Jul 28 18:49:54 client borg-backup: U /etc/shells
Jul 28 18:49:54 client borg-backup: U /etc/thermald/thermal-cpu-cdev-order.xml
Jul 28 18:49:54 client borg-backup: d /etc/thermald
Jul 28 18:49:54 client borg-backup: U /etc/UPower/UPower.conf
Jul 28 18:49:54 client borg-backup: d /etc/UPower
Jul 28 18:49:54 client borg-backup: d /etc
Jul 28 18:49:55 client borg-backup: ------------------------------------------------------------------------------
Jul 28 18:49:55 client borg-backup: Repository: ssh://borg@192.168.1.44/var/backup
Jul 28 18:49:55 client borg-backup: Archive name: etc-2026-07-28_18:49:53
Jul 28 18:49:55 client borg-backup: Archive fingerprint: 6631c2cfae5a09e80282954d421ad5b096b8a3ffd84cf3429b77048056560bebJul 28 18:49:55 client borg-backup: Time (start): Tue, 2026-07-28 18:49:54
Jul 28 18:49:55 client borg-backup: Time (end):   Tue, 2026-07-28 18:49:54
Jul 28 18:49:55 client borg-backup: Duration: 0.43 seconds
Jul 28 18:49:55 client borg-backup: Number of files: 734
Jul 28 18:49:55 client borg-backup: Utilization of max. archive size: 0%
Jul 28 18:49:55 client borg-backup: ------------------------------------------------------------------------------
Jul 28 18:49:55 client borg-backup:                        Original size      Compressed size    Deduplicated size
Jul 28 18:49:55 client borg-backup: This archive:                2.17 MB            975.53 kB                552 B
Jul 28 18:49:55 client borg-backup: All archives:                6.49 MB              2.91 MB              1.07 MB
Jul 28 18:49:55 client borg-backup: 
Jul 28 18:49:55 client borg-backup:                        Unique chunks         Total chunks
Jul 28 18:49:55 client borg-backup: Chunk index:                     693                 2139
Jul 28 18:49:55 client borg-backup: ------------------------------------------------------------------------------
Jul 28 18:49:57 client borg-backup: Keeping archive (rule: daily #1):        etc-2026-07-28_18:49:53              Tue, 2026-07-28 18:49:54 [6631c2cfae5a09e80282954d421ad5b096b8a3ffd84cf3429b77048056560beb]
Jul 28 18:49:57 client borg-backup: Pruning archive (1/1):                   etc-2026-07-28_18:44:44              Tue, 2026-07-28 18:44:45 [bd5865f284b01a78ab5f33037b39f774922340210bdb758409174fa494ac264d]
Jul 28 18:49:57 client borg-backup: Keeping archive (rule: daily[oldest] #2): etc-2026-07-28_14:56:10              Tue, 2026-07-28 14:56:23 [a5f225e6cf3b0d8087f94f3188f6aa20c98dc40d75d28ab9db80a58d73da9e7d]
Jul 28 18:49:58 client borg-backup: === Backup finished: Tue Jul 28 06:49:58 PM UTC 2026 ===
Jul 28 18:49:58 client systemd[1]: borg-backup.service: Deactivated successfully.
Jul 28 18:49:58 client systemd[1]: borg-backup.service: Consumed 1.364s CPU time.
```

Проверим, что таймер активировался и будет запускаться каждые 5 минут.
Выполните на client:

```bash
root@client:~# systemctl list-timers borg-backup.timer
NEXT                        LEFT          LAST                        PASSED       UNIT              ACTIVATES          
Tue 2026-07-28 18:54:52 UTC 3min 36s left Tue 2026-07-28 18:49:52 UTC 1min 23s ago borg-backup.timer borg-backup.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.
```
