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
replica-node ansible_host=192.168.0.104
```

также для того, чтобы поднять **кластер**, включающий в себя master и replica node, нам необходимо запустить ansible playbook, кроме этого в tasks будут прописаны определенные условия в конце блоков:

```bash
- hosts: app
  become: yes
  roles:
    - role: postgres
      tags: app

- hosts: master
  become: yes
  roles:
    - role: postgres
      tags: master

- hosts: replica
  become: yes
  roles:
    - role: postgres
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

```bash
ansible-galaxy init postgres
```

Все описание роли:

defaults/main.yml:

```bash
postgresql_user_name: postgres
postgresql_db_name: test

postgresql_version: "17"
postgresql_package: postgresql
postgresql_common_package: postgresql-common

pg_hba_conf_path: "/etc/postgresql/{{ postgresql_version }}/main/pg_hba.conf"
pg_conf_path: "/etc/postgresql/{{ postgresql_version }}/main/postgresql.conf"

postgresql_repo_url: "https://apt.postgresql.org/pub/repos/apt"
postgresql_repo_path: "/usr/share/postgresql-common/pgdg/apt.postgresql.org.sh"

postgresql_key_url: "https://www.postgresql.org/media/keys/ACCC4CF8.asc"
postgresql_key: "/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc"

postgresql_sources_list: "/etc/apt/sources.list.d/pgdg.list"
postgresql_data_dir: "/var/lib/postgresql/{{ postgresql_version }}/main"

postgresql_master_listen_addresses: '*'
postgresql_master_wal_level: replica
postgresql_master_archive_mode: on
postgresql_master_archive_command: '/bin/true'
postgresql_master_max_wal_senders: 5
postgresql_master_hot_standby: on
postgresql_replica_listen_addresses: '*'
postgresql_replica_wal_level: replica
postgresql_replica_archive_mode: on
postgresql_replica_archive_command: '/bin/true'
postgresql_replica_max_wal_senders: 5
postgresql_replica_hot_standby: on
master_host: 192.168.0.103
# master_host: master-node
replication_user: postgres
```
handlers/main.yml:

```bash
- name: Restart PostgreSQL service
  shell: |
    pg_ctlcluster {{ postgresql_version }} main restart
  args:
    executable: /bin/bash
  register: restart_result
  changed_when: "'server starting' in restart_result.stdout or 'restarted' in restart_result.stdout"
  failed_when: restart_result.rc != 0 and 'stopped' not in restart_result.stdout
```
Для удобства таски были разбиты на три файла

tasks/main.yml:

```bash
- name: Ensure Python is installed
  raw: test -e /usr/bin/python || (apt -y update && apt install -y python3)
  changed_when: false

- name: Install necessary system packages
  apt:
    name:
      - "{{ postgresql_common_package }}"
      - gnupg
      - curl
      - ca-certificates
      - python3-pip
      - acl
      - iputils-ping
    state: present
    update_cache: yes

- name: Install psycopg2-binary for app servers
  pip:
    name: psycopg2-binary
    state: present
    executable: /usr/bin/pip3

- name: Install PostgreSQL client on app servers
  apt:
    name: postgresql-client
    state: present
  when: "'app' in group_names"

# Задачи для master и replica
- name: Install PostgreSQL server and configure common settings
  block:
    - name: Run PostgreSQL PGDG script
      shell: "{{ postgresql_repo_path }} -y"
      args:
        creates: /etc/apt/sources.list.d/pgdg.list
      changed_when: false

    - name: Create directory for PostgreSQL repository key
      file:
        path: /usr/share/postgresql-common/pgdg
        state: directory
        mode: '0755'
      changed_when: false

    - name: Download the PostgreSQL repository signing key
      get_url:
        url: "{{ postgresql_key_url }}"
        dest: "{{ postgresql_key }}"
        mode: '0644'
      changed_when: false

    - name: Check if PGDG repository exists
      stat:
        path: "{{ postgresql_sources_list }}"
      register: pgdg_repo

    - name: Add PostgreSQL repository to sources list
      shell: |
        echo "deb [signed-by={{ postgresql_key }}] {{ postgresql_repo_url }} $(lsb_release -cs)-pgdg main" > {{ postgresql_sources_list }}
      when: not pgdg_repo.stat.exists
      changed_when: false

    - name: Update package list
      apt:
        update_cache: yes

    - name: Install PostgreSQL server
      apt:
        name: "{{ postgresql_package }}"
        state: present
        force: yes

    - name: Configure pg_hba.conf
      template:
        src: templates/pg_hba.conf
        dest: "{{ pg_hba_conf_path }}"
        owner: postgres
        group: postgres
        mode: '0644'
      notify:
        - Restart PostgreSQL service
  when: "'master' in group_names or 'replica' in group_names"

- import_tasks: master.yml
  when: "'master' in group_names"

- import_tasks: replica.yml
  when: "'replica' in group_names"
```

tasks/master.yml:

```bash
- name: Configure master node
  block:
    - name: Ensure PostgreSQL service is running on master
      shell: |
        pg_ctlcluster {{ postgresql_version }} main start
      args:
        executable: /bin/bash
      register: postgresql_start_master
      changed_when: "'server starting' in postgresql_start_master.stdout"
      failed_when: "'already running' not in postgresql_start_master.stdout and postgresql_start_master.rc != 0 and 'server starting' not in postgresql_start_master.stdout"

    - name: Configure postgresql.conf for Master
      blockinfile:
        path: "{{ pg_conf_path }}"
        block: |
          listen_addresses = '{{ postgresql_master_listen_addresses }}'
          wal_level = {{ postgresql_master_wal_level }}
          archive_mode = {{ postgresql_master_archive_mode }}
          archive_command = '{{ postgresql_master_archive_command }}'
          max_wal_senders = {{ postgresql_master_max_wal_senders }}
          hot_standby = {{ postgresql_master_hot_standby }}
      changed_when: false

    - name: Create PostgreSQL database
      become_user: postgres
      postgresql_db:
        name: "{{ postgresql_db_name }}"
      register: db_creation
      changed_when: db_creation.changed

    - name: Create PostgreSQL user
      become_user: postgres
      postgresql_user:
        name: "{{ postgresql_user_name }}"
      register: user_creation
      changed_when: user_creation.changed

    - name: Restart PostgreSQL service on master
      shell: |
        pg_ctlcluster {{ postgresql_version }} main restart
      args:
        executable: /bin/bash
      register: restart_result
      changed_when: "'server starting' in restart_result.stdout or 'restarted' in restart_result.stdout"
      failed_when: "'stopped' in restart_result.stdout or restart_result.rc != 0"
```

tasks/replica.yml:

```bash
- name: Configure replica node
  block:
  - name: Check PostgreSQL service status on replica
    shell: |
      pg_lsclusters | grep "{{ postgresql_version }}" | grep main | awk '{print $3}'
    args:
      executable: /bin/bash
    register: replica_status
    changed_when: false

  - name: Stop PostgreSQL service for replica if running
    shell: |
      pg_ctlcluster {{ postgresql_version }} main stop
    args:
      executable: /bin/bash
    register: replica_stop
    changed_when: "'server stopped' in replica_stop.stdout"
    failed_when: "'is not running' not in replica_stop.stdout and replica_stop.rc != 0"
    when: replica_status.stdout.strip() == "online"

  - name: Check if PostgreSQL data directory exists
    stat:
      path: "{{ postgresql_data_dir }}"
    register: data_dir_status
    become_user: postgres

  - name: Remove existing PostgreSQL data and create new directory
    shell: |
      rm -rf {{ postgresql_data_dir }} &&
      mkdir -p {{ postgresql_data_dir }} &&
      chmod go-rwx {{ postgresql_data_dir }}
    args:
      executable: /bin/bash
    become_user: postgres
    when: data_dir_status.stat.exists
    changed_when: false

  - name: Check if standby.signal exists
    stat:
      path: "{{ postgresql_data_dir }}/standby.signal"
    register: standby_signal_check
    become_user: postgres

  - name: Perform base backup from Master to Replica
    command: >
      pg_basebackup -P -R -X stream -c fast -h {{ master_host }} -U {{ replication_user }} -D "{{ postgresql_data_dir }}"
    args:
      creates: "{{ postgresql_data_dir }}/standby.signal"
    become_user: postgres
    when: not standby_signal_check.stat.exists
    changed_when: false

  - name: Configure postgresql.conf for Replica
    blockinfile:
      path: "{{ pg_conf_path }}"
      block: |
        primary_conninfo = 'host={{ master_host }} port=5432 user={{ replication_user }}'
        hot_standby = on
    changed_when: false

  - name: Ensure no existing PostgreSQL processes conflict on replica
    shell: |
      ps aux | grep '[p]ostgres' | awk '{print $2}' | xargs -r kill -9
      ps aux | grep '[p]ostgres' | wc -l
    args:
      executable: /bin/bash
    register: postgres_kill
    failed_when: false
    changed_when: false
  
  - name: Start PostgreSQL service for replica
    shell: |
      pg_ctlcluster {{ postgresql_version }} main start
    args:
      executable: /bin/bash
    register: replica_start
    changed_when: "'server starting' in replica_start.stdout"
    failed_when: "'already running' not in replica_start.stdout and 'port conflict' not in replica_start.stderr and replica_start.rc != 0"
```
Также выносим переменки в inventry/group_vars для переопределения:

master.yml:

```bash
postgresql_master_listen_addresses: '*'
postgresql_master_wal_level: replica
postgresql_master_archive_mode: on
postgresql_master_archive_command: '/bin/true'
postgresql_master_max_wal_senders: 5
postgresql_master_hot_standby: on
```

replica.yml:

```bash
postgresql_replica_listen_addresses: '*'
postgresql_replica_wal_level: replica
postgresql_replica_archive_mode: on
postgresql_replica_archive_command: '/bin/true'
postgresql_replica_max_wal_senders: 5
postgresql_replica_hot_standby: on
master_host: 192.168.0.103
# master_host: master-node
replication_user: postgres
```

db.yml:

```bash
postgresql_user_name: postgres
postgresql_db_name: test

postgresql_version: "17"
postgresql_package: postgresql
postgresql_common_package: postgresql-common

pg_hba_conf_path: "/etc/postgresql/{{ postgresql_version }}/main/pg_hba.conf"
pg_conf_path: "/etc/postgresql/{{ postgresql_version }}/main/postgresql.conf"

postgresql_repo_url: "https://apt.postgresql.org/pub/repos/apt"
postgresql_repo_path: "/usr/share/postgresql-common/pgdg/apt.postgresql.org.sh"

postgresql_key_url: "https://www.postgresql.org/media/keys/ACCC4CF8.asc"
postgresql_key: "/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc"

postgresql_sources_list: "/etc/apt/sources.list.d/pgdg.list"
postgresql_data_dir: "/var/lib/postgresql/{{ postgresql_version }}/main"
```

Теперь проверим molecule test, что они отрабатывают без ошибок:

Converge:

<img width="666" alt="image" src="https://github.com/user-attachments/assets/a551c55e-ed1a-47e2-aaef-bb4f32f58492" />
<img width="919" alt="image" src="https://github.com/user-attachments/assets/688e2d6d-525e-42e9-bed3-6a70a7f48f35" />

теперь тест на Idempotence:

<img width="650" alt="image" src="https://github.com/user-attachments/assets/cae9f7d6-5a88-4aa3-9c1c-953f667f1418" />
<img width="924" alt="image" src="https://github.com/user-attachments/assets/44b84edd-49a9-4600-9945-4fcc34dd7dc3" />

Destroy:
<img width="1233" alt="image" src="https://github.com/user-attachments/assets/ca6773e7-738f-4712-a901-366ca44c5d6c" />

После прогонки молекул тестов, запускаем playbook:

```bash
ansible-playbook -i inventory/hosts -u vagrant /home/vagrant/playbooks/postgres_playbook
```
Конечный результат всех тасок:

<img width="924" alt="image" src="https://github.com/user-attachments/assets/5899a483-58d0-4854-baea-e968023d24b9" />

## 3. Добавить возможность смены директории с данными на кастомную

Для смены директории, в которой будут храниться данные, пропишем данную переменку:

```bash
postgresql_data_dir: "/var/lib/postgresql/{{ postgresql_version }}/main"
```
## 4. Добавить возможность создания баз данных и пользователей

Заходим на мастер и проверяем БД, можно увидеть, что была успешно создана БД test:

<img width="1005" alt="image" src="https://github.com/user-attachments/assets/fea3528b-67cb-4368-bf46-c11949b770c6" />

Также добавим нового пользователя:

Юзеры до добавления нового:
<img width="850" alt="image" src="https://github.com/user-attachments/assets/0566e82a-a13a-44e3-b06a-bdb893888c78" />

Создаем нового, выдаем права, смотрим, что пользователь появился в таблице:
<img width="425" alt="image" src="https://github.com/user-attachments/assets/6265125c-242e-45ae-b8cc-2032fdb77bb9" />
<img width="490" alt="image" src="https://github.com/user-attachments/assets/432d03f8-9120-4b0f-ae7f-606646ae1b49" />
<img width="837" alt="image" src="https://github.com/user-attachments/assets/b67bd135-8e34-46c7-a147-f6e778876ecb" />

## 5. Добавить функционал настройки streaming-репликации***


## 6. Продумать логику определения master и replica нод СУБД и их настройки при работе роли***

