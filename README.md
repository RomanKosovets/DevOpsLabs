# Лабораторная работа №3
## 1. Поднять Vagrant-окружение при помощи файла Vagrantfile, используя команду vagrant up

На первом этапе было поднято Vagrant-окружение:

<img width="566" alt="image" src="https://github.com/user-attachments/assets/bff3378f-8655-4235-a0ef-8620c193c098">
 
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

Также некоторые переменные были вынесены в папку default:

<img width="470" alt="image" src="https://github.com/user-attachments/assets/bbd7294e-a047-40db-8227-4a741913a146">


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
``

