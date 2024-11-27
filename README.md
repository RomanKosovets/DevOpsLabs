# Лабораторная работа №2
## 1. Инициировать структуру роли “**Docker**” через Ansible-Galaxy

Инициирование структуры роли в Ansible Galaxy — это стандартный и удобный способ создать шаблон для роли, обеспечивающий правильную организацию файлов и каталогов. Для роли с названием "Docker" будем использовать слудующую команду: 

![image](https://github.com/user-attachments/assets/9a100836-06ae-4143-a3a0-10b8897e0394)

В итоге мы имеем следующую структуру роли: 

![image](https://github.com/user-attachments/assets/46048059-75ad-46f7-a0e6-c8c353b3d2d4)

Каждый файл и папка имеет своё предназначение:

- defaults/ — переменные по умолчанию
- files/ — статические файлы, которые могут быть скопированы на удалённые хосты
- handlers/main.yml — основной файл для определения обработчико
- meta/main.yml — мета-информация о роли, включая зависимости (если эта роль зависит от других ролей)
- tasks/main.yml — основной каталог, где описываются шаги выполнения роли
- templates/ — шаблоны файлов (например, конфигурации), которые обрабатываются через Jinja2
- tests/ — содержит тестовые сценарии для проверки роли
- vars/ — переменные с более высоким приоритетом. Не рекомендуется хранить секреты
- README.md — описание роли (документация)

## 2. Вынести из предыдущего плейбука задачи установки Docker в отдельную роль
Используя плейбук из первой ЛР, где описаны задачи установки Docker, перемещаем соответствующие задачи в файл tasks/main.yml роли docker

```bash
- name: Install required system packages
  apt:
    pkg:
    - python3-pip
    state: latest
    update_cache: true

- name: Install Docker SDK for Python
  pip:
    name: docker
    state: present
    executable: pip3

- name: Add Docker GPG apt Key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker Repository
  apt_repository:
    repo: deb https://download.docker.com/linux/ubuntu focal stable
    state: present

- name: Update apt and install docker-ce
  apt:
    name: docker-ce
    state: latest
    update_cache: true

- name: Add current user to the docker group
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes
```

## 3. Параметризовать роль через переменные
## 4. Создать в gitlab группу для репозиториев с ролями, запушить роль “**Docker**” в репозиторий
## 5. Создать файл requirements.yml для установки роли Docker из репозитория
## 6. Запустить приложение на нодах группы [app] используя ansible-playbook с ролью “**Docker**”
