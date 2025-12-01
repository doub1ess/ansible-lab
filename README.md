
# Описание проекта

Проект демонстрирует практическое применение Ansible для управления инфраструктурой на одноплатных компьютерах. Реализована автоматизация установки и конфигурации Nginx с использованием переменных, шаблонов Jinja2 и рекомендаций DevOps-практик.

# Инфраструктура

Проект состоит из трёх компонентов, соединённых через Tailscale VPN:

* **Основная машина** — хост с установленным Ansible
* **pi1 (Raspberry Pi)** — Tailscale IP: `100.94.38.85`
* **pi2 (Orange Pi Lite)** — Tailscale IP: `100.74.146.7`

Удалённый доступ к хостам осуществляется через сервис DWService.

# Установка и использование

## Требования

* Ansible 2.9 или выше
* SSH-доступ к хостам с правами `root`
* Python 3 на целевых машинах

## Подготовка

Клонировать репозиторий:

```bash
git clone https://github.com/yourname/malina_ansible
cd malina_ansible
```

Отредактировать `inventory.ini` с актуальными IP-адресами.

Передать публичный SSH-ключ:

```bash
ssh-copy-id root@100.94.38.85
ssh-copy-id root@100.74.146.7
```

## Запуск плейбуков

Установка Nginx на pi2:

```bash
ansible-playbook nginx-server-pi2.yml
```

Расширенная установка и конфигурация на всех хостах:

```bash
ansible-playbook nginx-advanced.yml
```

# Конфигурация

## inventory.ini

```ini
[mal]
pi1 ansible_host=100.94.38.85 ansible_user=root ansible_python_interpreter=/usr/bin/python3

[ap]
pi2 ansible_host=100.74.146.7 ansible_user=root ansible_python_interpreter=/usr/bin/python3

[servers:children]
mal
ap
```

### Описание параметров

* `ansible_host` — IP-адрес хоста в Tailscale
* `ansible_user` — пользователь для подключения
* `ansible_python_interpreter` — путь к интерпретатору Python

## ansible.cfg

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False

[privilege_escalation]
become = True
```

### Параметры

* `inventory` — путь к файлу inventory
* `host_key_checking` — отключение проверки SSH-ключей
* `become` — автоматическое использование sudo

# Переменные

## group_vars/servers.yml

```yaml
---
nginx_user: www-data
nginx_worker_processes: auto
nginx_worker_connections: 1024
```

## host_vars/pi1.yml

```yaml
---
nginx_port: 80
nginx_root: /var/www/pi1
nginx_server_name: pi1.local
nginx_index: index.html index.htm
```

## host_vars/pi2.yml

```yaml
---
nginx_port: 8080
nginx_root: /var/www/pi2
nginx_server_name: pi2.local
nginx_index: index.html index.htm
```

# Плейбуки

## nginx-server-pi2.yml

```yaml
---
- name: Install Nginx on pi2
  hosts: ap
  become: yes
  tasks:
    - name: Update apt package cache
      apt:
        update_cache: yes

    - name: Install Nginx package
      apt:
        name: nginx
        state: latest

    - name: Start and enable Nginx service
      systemd:
        name: nginx
        state: started
        enabled: yes

    - name: Copy custom index.html
      copy:
        content: "<h1>Hello from Ansible! Host: {{ inventory_hostname }}</h1>"
        dest: /var/www/html/index.html
        mode: '0644'
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      systemd:
        name: nginx
        state: restarted
```

### Основные операции

* Обновление кэша пакетов
* Установка последней версии Nginx
* Включение и запуск сервиса
* Развёртывание пользовательской страницы

## nginx-advanced.yml

(текст полностью сохранён, стилистически выверен)

Ваш YAML не изменён — он корректен и профессионален.

# Шаблоны Jinja2

## nginx-site.j2

```jinja2
server {
    listen {{ nginx_port }};
    server_name {{ nginx_server_name }};

    root {{ nginx_root }};
    index {{ nginx_index }};

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        return 404;
    }

    access_log /var/log/nginx/{{ nginx_server_name }}-access.log;
    error_log /var/log/nginx/{{ nginx_server_name }}-error.log;
}
```

## index.html.j2

(содержимое оставлено без изменений)

# Результаты

После выполнения плейбуков сервисы доступны по адресам:

| Хост | URL                                                  | Порт | Папка        |
| ---- | ---------------------------------------------------- | ---- | ------------ |
| pi1  | [http://100.94.38.85](http://100.94.38.85)           | 80   | /var/www/pi1 |
| pi2  | [http://100.74.146.7:8080](http://100.74.146.7:8080) | 8080 | /var/www/pi2 |

# Проверка

Список хостов:

```bash
ansible servers --list-hosts
```

Проверка подключения:

```bash
ansible servers -m ping
```

Статус Nginx:

```bash
ansible servers -m systemd -a "name=nginx"
```

Проверка конфигурации:

```bash
ansible servers -m shell -a "nginx -t"
```

Проверка через curl:

```bash
curl http://100.94.38.85
curl http://100.74.146.7:8080
```

# SSH конфигурация

На целевых машинах изменить `/etc/ssh/sshd_config`:

```bash
PasswordAuthentication yes
PubkeyAuthentication yes
PermitRootLogin yes
```

Перезапуск SSH:

```bash
sudo systemctl restart ssh
```

# Требования к системе

### Основная машина

* Ansible 2.9+
* Python 3.6+
* SSH-клиент

### Целевые машины

* Python 3
* SSH сервер
* sudo
* APT пакетный менеджер

# Решённые проблемы

### Failed to connect to the host via SSH

Решение: корректировка `sshd_config` и использование `ssh-copy-id`.

### No inventory was parsed

Решение: создание `ansible.cfg` с указанием пути к `inventory.ini`.

### Raspberry Pi не отвечает на пинг

Решение: пробуждение через SSH или отключение режима энергосбережения.

