Описание проекта

Проект демонстрирует практическое применение Ansible для управления инфраструктурой на одноплатных компьютерах. Реализована полная автоматизация установки и конфигурации Nginx с использованием переменных, шаблонов Jinja2 и best practices DevOps.
Инфраструктура

Проект состоит из трёх компонентов, соединённых через Tailscale VPN:

    Основная машина: Компьютер с Ansible для управления инфраструктурой

    pi1 (Raspberry Pi): Одноплатный компьютер, Tailscale IP 100.94.38.85

    pi2 (Orange Pi Lite): Одноплатный компьютер, Tailscale IP 100.74.146.7

Доступ к хостам предоставлен через сервис dwservice.
Структура проекта

text
malina_ansible/
├── ansible.cfg                 # Конфигурация Ansible
├── inventory.ini              # Описание хостов и групп
├── nginx-server-pi2.yml       # Простой плейбук для установки Nginx
├── nginx-advanced.yml         # Продвинутый плейбук с Jinja2
├── group_vars/
│   └── servers.yml            # Общие переменные для всех серверов
├── host_vars/
│   ├── pi1.yml                # Переменные для Raspberry Pi
│   └── pi2.yml                # Переменные для Orange Pi Lite
├── templates/
│   ├── nginx-site.j2          # Шаблон конфигурации Nginx
│   └── index.html.j2          # Шаблон главной страницы
└── README.md                  # Этот файл

Установка и использование
Требования

    Ansible 2.9 или выше

    SSH доступ к хостам с правами root

    Python 3 на целевых машинах

Подготовка

    Клонировать репозиторий:

bash
git clone https://github.com/yourname/malina_ansible
cd malina_ansible

    Отредактировать inventory.ini с актуальными IP-адресами.

    Передать публичный SSH ключ:

bash
ssh-copy-id root@100.94.38.85
ssh-copy-id root@100.74.146.7

Запуск плейбуков

Простая установка Nginx на pi2:

bash
ansible-playbook nginx-server-pi2.yml

Продвинутая установка с конфигурацией для обеих машин:

bash
ansible-playbook nginx-advanced.yml

Конфигурация
inventory.ini

text
[mal]
pi1 ansible_host=100.94.38.85 ansible_user=root ansible_python_interpreter=/usr/bin/python3

[ap]
pi2 ansible_host=100.74.146.7 ansible_user=root ansible_python_interpreter=/usr/bin/python3

[servers:children]
mal
ap

Описание параметров:

    ansible_host: IP адрес хоста в Tailscale сети

    ansible_user: пользователь для подключения (root)

    ansible_python_interpreter: путь к интерпретатору Python

ansible.cfg

text
[defaults]
inventory = inventory.ini
host_key_checking = False

[privilege_escalation]
become = True

Параметры:

    inventory: путь к файлу хостов

    host_key_checking: отключение проверки SSH ключей

    become: автоматическое использование sudo

Переменные
group_vars/servers.yml

text
---
nginx_user: www-data
nginx_worker_processes: auto
nginx_worker_connections: 1024

host_vars/pi1.yml

text
---
nginx_port: 80
nginx_root: /var/www/pi1
nginx_server_name: pi1.local
nginx_index: index.html index.htm

host_vars/pi2.yml

text
---
nginx_port: 8080
nginx_root: /var/www/pi2
nginx_server_name: pi2.local
nginx_index: index.html index.htm

Плейбуки
nginx-server-pi2.yml

Простой плейбук для установки Nginx на одной машине:

text
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

Выполняемые операции:

    Обновление кэша пакетов APT

    Установка Nginx в последней версии

    Запуск и включение сервиса в автозапуск

    Развёртывание пользовательской страницы index.html

nginx-advanced.yml

Продвинутый плейбук с использованием шаблонов Jinja2:

text
---
- name: Deploy Advanced Nginx with Jinja2
  hosts: servers
  become: yes
  vars:
    nginx_config_dir: /etc/nginx/sites-available

  tasks:
    - name: Clean previous repos (fix apt)
      shell: |
        sed -i '/backports/d' /etc/apt/sources.list
        rm -f /etc/apt/sources.list.d/*.list
      ignore_errors: yes

    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install nginx
      apt:
        name: nginx
        state: latest

    - name: Create web roots
      file:
        path: "{{ nginx_root }}"
        state: directory
        mode: '0755'
        owner: "{{ nginx_user }}"
        group: "{{ nginx_user }}"

    - name: Deploy custom index.html
      template:
        src: index.html.j2
        dest: "{{ nginx_root }}/index.html"
        mode: '0644'
        owner: "{{ nginx_user }}"
        group: "{{ nginx_user }}"
      notify:
        - Test nginx config
        - Restart nginx

    - name: Deploy nginx site config from template
      template:
        src: nginx-site.j2
        dest: "{{ nginx_config_dir }}/{{ inventory_hostname }}"
        mode: '0644'
      notify:
        - Test nginx config
        - Restart nginx

    - name: Enable site (symlink)
      file:
        src: "{{ nginx_config_dir }}/{{ inventory_hostname }}"
        dest: /etc/nginx/sites-enabled/{{ inventory_hostname }}"
        state: link
      notify:
        - Test nginx config
        - Restart nginx

    - name: Disable default site
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify:
        - Test nginx config
        - Restart nginx

    - name: Start and enable nginx
      systemd:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Test nginx config
      shell: nginx -t
      register: result
      changed_when: False

    - name: Restart nginx
      systemd:
        name: nginx
        state: restarted

Выполняемые операции:

    Очистка проблемных репозиториев APT

    Установка Nginx на всех хостах группы servers

    Создание отдельных корневых папок для каждого хоста

    Генерация конфигурации Nginx из шаблона

    Создание symlink для активации сайтов

    Отключение конфигурации по умолчанию

    Проверка синтаксиса перед перезапуском

    Перезапуск только при изменениях

Шаблоны Jinja2
templates/nginx-site.j2

text
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

Переменные подставляются из host_vars/group_vars.
templates/index.html.j2

xml
<!DOCTYPE html>
<html>
<head>
    <title>{{ inventory_hostname }} - Ansible Nginx</title>
    <style>
        body { 
            font-family: Arial; 
            text-align: center; 
            margin-top: 100px; 
            background: #f0f8ff; 
        }
        h1 { color: #2c3e50; }
        .info { 
            margin: 20px; 
            padding: 20px; 
            background: white; 
            border-radius: 10px; 
            display: inline-block; 
        }
    </style>
</head>
<body>
    <div class="info">
        <h1>Hello from {{ inventory_hostname }}!</h1>
        <p><strong>Port:</strong> {{ nginx_port }}</p>
        <p><strong>Root:</strong> {{ nginx_root }}</p>
        <p><strong>OS:</strong> {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
        <p><strong>Uptime:</strong> {{ ansible_uptime_seconds }} seconds</p>
    </div>
</body>
</html>

Результаты

После выполнения плейбуков доступны следующие адреса:
Хост	Адрес	Порт	Корневая папка
pi1	http://100.94.38.85	80	/var/www/pi1
pi2	http://100.74.146.7:8080	8080	/var/www/pi2
Проверка

Список доступных хостов:

bash
ansible servers --list-hosts

Проверка подключения:

bash
ansible servers -m ping

Статус Nginx:

bash
ansible servers -m systemd -a "name=nginx"

Проверка конфигурации:

bash
ansible servers -m shell -a "nginx -t"

Проверка через curl:

bash
curl http://100.94.38.85
curl http://100.74.146.7:8080

SSH конфигурация

На целевых машинах требуется отредактировать /etc/ssh/sshd_config:

bash
PasswordAuthentication yes
PubkeyAuthentication yes
PermitRootLogin yes

Перезапустить SSH:

bash
sudo systemctl restart ssh

Требования к системе

На основной машине:

    Ansible 2.9 или выше

    Python 3.6 или выше

    SSH клиент

На целевых машинах:

    Python 3

    SSH сервер

    Sudo

    APT пакетный менеджер

Решённые проблемы

Проблема: Failed to connect to the host via ssh
Решение: Настройка /etc/ssh/sshd_config и ssh-copy-id

Проблема: No inventory was parsed, only implicit localhost is available
Решение: Создание ansible.cfg с указанием пути к inventory.ini

Проблема: Raspberry Pi не отвечает на пинги
Решение: Пробуждение через SSH или отключение спящего режима
