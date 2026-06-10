# Домашнее задание "Ansible"

## Задание

Подготовить стенд как минимум с одним сервером. 
На этом сервере, используя Ansible, необходимо развернуть nginx со следующими условиями:
1. необходимо использовать модуль yum/apt;
2. конфигурационные файлы должны быть взяты из шаблона jinja2 с переменными;
3. после установки nginx должен быть в режиме enabled в systemd;
4. должен быть использован notify для старта nginx после установки;
5. сайт должен слушать на нестандартном порту — 8080, для этого использовать переменные в Ansible.
---

## Выполнение задания

Установим Ansible:
```bash
root@client:/home/user# sudo apt update
root@client:/home/user# sudo apt install ansible
root@client:/home/user# ansible --version
ansible [core 2.16.3]
  config file = None
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /root/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.12.3 (main, Mar 23 2026, 19:04:32) [GCC 13.3.0] (/usr/bin/python3)
  jinja version = 3.1.2
  libyaml = True


```
Создадим дирректорию для работы с ansible:
```bash
root@client:/home/user# mkdir nginx-ansible
```

Создадим ansible.cfg с минимальными настройками:
```bash
root@client:/home/user# touch ansible.cfg
root@client:/home/user# nano ansible.cfg

[defaults]
host_key_checking = False
inventory = inventory.ini
```

Создадим inventory.ini
```bash
root@client:/home/user# touch inventory.ini
root@client:/home/user# nano inventory.ini

[webserver]
localhost ansible_connection=local

[webserver:vars]
ansible_python_interpreter=/usr/bin/python3
become=yes
```

Создадим group_vars/all.yml
```bash
root@client:/home/user# mkdir group_vars
root@client:/home/user# touch group_vars/all.yml
root@client:/home/user# nano group_vars/all.yml

---
nginx_port: 8080
server_name: _
root_directory: /var/www/html
```

Создадим templates/nginx.conf.j2

```bash
root@client:/home/user# mkdir templates
root@client:/home/user# touch templates/nginx.conf.j2
root@client:/home/user# nano templates/nginx.conf.j2

server {
    listen {{ nginx_port }} default_server;
    listen [::]:{{ nginx_port }} default_server;

    root {{ root_directory }};
    index index.html index.htm index.nginx-debian.html;

    server_name {{ server_name }};

    location / {
        try_files $uri $uri/ =404;
    }
}

```

Создадим playbook.yml

```bash
root@client:/home/user# touch playbook.yml
root@client:/home/user# nano playbook.yml

---
- name: Установка и настройка Nginx с шаблоном Jinja2 и notify
  hosts: webserver
  become: yes
  gather_facts: yes

  tasks:
    # 1. Установка nginx через apt (модуль apt)
    - name: Установить nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes
      notify: start_and_enable_nginx

    # 2. Применить шаблон Jinja2 с переменной порта
    - name: Создать конфигурацию сайта из шаблона
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
        owner: root
        group: root
        mode: '0644'
        backup: yes
      notify: reload_nginx

    # 3. Включить сайт (симлинк)
    - name: Включить сайт в sites-enabled
      ansible.builtin.file:
        src: /etc/nginx/sites-available/default
        dest: /etc/nginx/sites-enabled/default
        state: link
      notify: reload_nginx

    # 4. Запустить nginx и добавить в автозагрузку (enabled в systemd)
    - name: Запустить nginx и включить автозапуск
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    # Обработчик, вызываемый после установки (выполняет start + enable)
    - name: start_and_enable_nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

    # Обработчик для перезагрузки конфигурации без остановки
    - name: reload_nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded

```


Проверяем синтаксис плейбука
```bash
root@client:/home/user# ansible-playbook playbook.yml --syntax-check

playbook: playbook.yml
```

Поверяем подключение:
```bash
root@client:/home/user# ansible all -m ping
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
Запускаем плейбук

```bash
root@client:/home/user# ansible-playbook playbook.yml

PLAY [Установка и настройка Nginx с шаблоном Jinja2 и notify] ********************************************************************************************************************
TASK [Gathering Facts] ***********************************************************************************************************************************************************ok: [localhost]

TASK [Установить nginx] **********************************************************************************************************************************************************ok: [localhost]

TASK [Создать конфигурацию сайта из шаблона] *************************************************************************************************************************************changed: [localhost]

TASK [Включить сайт в sites-enabled] *********************************************************************************************************************************************ok: [localhost]

TASK [Запустить nginx и включить автозапуск] *************************************************************************************************************************************ok: [localhost]

RUNNING HANDLER [reload_nginx] ***************************************************************************************************************************************************changed: [localhost]

PLAY RECAP ***********************************************************************************************************************************************************************localhost                  : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

Проверяем результат:
```bash
root@client:/home/user# curl http://localhost:8080
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
...

```

Проверяем статуус службы:
```bash
root@client:/home/user# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-06-10 19:24:22 UTC; 33min ago
       Docs: man:nginx(8)
    Process: 2073 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 2093 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 3978 ExecReload=/usr/sbin/nginx -g daemon on; master_process on; -s reload (code=exited, status=0/SUCCESS)
   Main PID: 2097 (nginx)
      Tasks: 2 (limit: 1056)
     Memory: 3.2M (peak: 4.3M)
        CPU: 29ms
     CGroup: /system.slice/nginx.service
             ├─2097 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             └─3980 "nginx: worker process"

Jun 10 19:24:20 client systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Jun 10 19:24:22 client systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
Jun 10 19:54:59 client systemd[1]: Reloading nginx.service - A high performance web server and a reverse proxy server...
```

Dсё выполнено в полном соответствии с планом и техническим заданием.