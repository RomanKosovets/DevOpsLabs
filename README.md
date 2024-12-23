# Лабораторная работа №2
## 1. Развернуть виртуальную машину для приложения (app) и виртуальную машину для будущей системы базы данных (db) через Vagrantfile

Для этого поправим файл hosts в inventory добавив нужные хосты и группы:
1. [app] для приложения
2. [master] - для master node
3. [replica] - для replica node
4. [db] - для переопределения переменок
   
```bash
[app]
worker2 ansible_host=192.168.0.102

[web]
worker1 ansible_host=192.168.0.101
worker2 ansible_host=192.168.0.102

[vpn]
worker1 ansible_host=192.168.0.101

[db]
worker4 ansible_host=192.168.0.104

[master]
master-node ansible_host=192.168.0.103

[replica]
replica-node ansoble_host=192.168.0.104
```

также для того, чтобы поднять кластер, включающий в себя master и replica node, нам необходимо запустить ansible playbook, кроме этого в tasks будут прописаны определенные условия в конце блоков:

```bash
- hosts: app
  become: yes
  roles:
    - role: postgresql
      tags: app

- hosts: master
  become: yes
  roles:
    - role: postgresql
      tags: master

- hosts: replica
  become: yes
  roles:
    - role: postgresql
      tags: replica
```

Условия для правильной настройки, которые позволяют выполнять задачи только для определенных хостов, так как настройка каждого отличается:

```bash
when: "'app' in group_names"
when: "'master' in group_names or 'replica' in group_names"
when: "'master' in group_names"
when: "'replica' in group_names"
```

## 2. Написать роль установки PostgreSQL и плейбук, который ее разворачивает, вынести все параметры в переменные, все переменные должны быть префиксованы, значения переменных устанавливаются через group_vars для плейбука, роль должна быть покрыта тестами:

Создадим роль postgres:


## 3. Добавить возможность смены директории с данными на кастомную
## 4. Добавить возможность создания баз данных и пользователей
## 5. Добавить функционал настройки streaming-репликации***
## 6. Продумать логику определения master и replica нод СУБД и их настройки при работе роли***
