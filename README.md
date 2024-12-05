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

<img width="348" alt="image" src="https://github.com/user-attachments/assets/384001d7-a8ea-48b1-a3c1-a07a178bcf9c">

Также некоторые переменные были вынесены в папку default:

<img width="470" alt="image" src="https://github.com/user-attachments/assets/bbd7294e-a047-40db-8227-4a741913a146">


## 4. Создать в gitlab группу для репозиториев с ролями, запушить роль “**Docker**” в репозиторий

В gitlab была создана группа и запушена роль Docker в репозиторий

<img width="1256" alt="image" src="https://github.com/user-attachments/assets/28a571a3-e516-46c1-83de-28b008efe985">
<img width="481" alt="image" src="https://github.com/user-attachments/assets/5e9ac1f0-2cbc-4a06-a73a-5959195a895d">

## 5. Создать файл requirements.yml для установки роли Docker из репозитория

Создадим файл для автоматической установки роли Docker из репозитория, где указан источник, а также используем команду для установки роли

<img width="822" alt="image" src="https://github.com/user-attachments/assets/5e4dac73-8075-46f3-8a45-0f52e2a80b24">

## 6. Запустить приложение на нодах группы [app] используя ansible-playbook с ролью “**Docker**”

Запускаем ansible-playbook с ролью Docker на нодах с помощью команды:

<img width="954" alt="image" src="https://github.com/user-attachments/assets/64f629e3-1851-455a-b7d1-5b3b928f5853">

