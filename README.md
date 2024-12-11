# Лабораторная работа №4
## 1. Написать Ansible Playbook поднимающий OpenVPN сервер / Через Molecule инициировать структуру для роли и вынести задачи поднятия сервера, с конфигурацией через переменные

Для начала установим molecule:
```bash
pip3 install molecule --break-system-packages
```

<img width="1312" alt="image" src="https://github.com/user-attachments/assets/b21568a8-2d5b-4b49-bee5-b690c0acab0f">

На первом этапе была создана роль данной командой:
```bash
ansible-galaxy init openvpn
```

После этого проинициализируем molecule в роле, что добавит нам необходимые папки с правильной структурой:
```bash
molecule init scenario
```

Далее напишем playbook

 
## 2. Написать роль для установки nginx, запушить в репозиторий и добавить в requirements.yml

Для создания роли "Nginx" использовалась команда, которая автоматически создает шаблон каталогов и файлов для роли, что помогает структурировать наши роли 

```bash
ansible-galaxy init Nginx
```

<img width="494" alt="image" src="https://github.com/user-attachments/assets/81428b32-cfed-465c-9aa6-59568e91a2f6">

далее были написаны tasks и вынесены в новую роль Nginx в файл ```Nginx/tasks/main.yml```

```bash
---
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Copy nginx site configuration
  template:
    src: "{{ nginx_site_template }}"
    dest: "{{ nginx_site_available }}"
  notify: Restart nginx

- name: Enable nginx site
  file:
    src: "{{ nginx_site_available }}"
    dest: "{{ nginx_site_enabled }}"
    state: link

- name: Ensure nginx is running
  service:
    name: nginx
    state: started
    enabled: true
```

Некоторые переменные были вынесены в папку default:

<img width="470" alt="image" src="https://github.com/user-attachments/assets/bbd7294e-a047-40db-8227-4a741913a146">

также меняем requirements.yml, прописывая путь до роли с гитлаба

```bash
---
roles:
  - name: Docker
    src: https://gitlab.com/itmo-devops-ansible-roles/labs/-/archive/main/labs-main.tar
    version: main
  - name: nginx
    src: https://gitlab.com/itmo-devops-ansible-roles/labs/-/archive/main/labs-main.tar
    version: main
```

Пушим изменения на гитлаб, где у нас лежала первая роль Docker:

<img width="1355" alt="image" src="https://github.com/user-attachments/assets/5ccb39d5-5fa0-4315-b96e-ec116b999158">


## 3. Подготовить в инвентори шаблон конфигурационного файла для Nginx и для default-сайта, с настройками для отдачи статических файлов. Учесть в шаблоне проксирование динамики в контейнер с Django


Добавлен файл nginx.conf.j2 в ```Nginx/templates/nginx.conf.j2``` и также для переопределения добавлен в файл ```inventory/templates/nginx.conf.j2```, как и default переменки для template

```bash
server {
    listen 80;
    server_name localhost;

    location /static/ {
        alias /home/{{ ansible_user }}/app/catalog/static/;
    }

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Дальше был написан простой playbook для nginx:

```bash
---
- hosts: web
  become: yes

  roles:
    - nginx
```

После настройки запускаем playbook:

<img width="828" alt="image" src="https://github.com/user-attachments/assets/c1eade9a-a2ea-406e-80b4-e234c4b29045">

Заходим на воркер и проверяем доступность nginx:

<img width="374" alt="image" src="https://github.com/user-attachments/assets/1d657884-8de2-4825-81be-debd8c7df336">

Заходим в nginx, все работает:

<img width="817" alt="image" src="https://github.com/user-attachments/assets/176c985c-f5ec-4881-bddb-c59e44557471">


