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

<img width="671" alt="image" src="https://github.com/user-attachments/assets/9a97db59-6954-4eea-a3d2-4f9f879ca867" />


Далее напишем playbook, а также опишем роль openvpn:

playbook:

```bash
---
- name: Set up OpenVPN Server
  hosts: vpn
  become: yes
  roles:
    - openvpn
 ```

default/main.yml и vars/main.yml переменки для шаблонов сервера и клиента: 

```bash
openvpn_packages:
  - openvpn
  - easy-rsa

openvpn_config_path: /etc/openvpn/
easy_rsa_path: /etc/openvpn/easy-rsa
easy_rsa_pki_path: /etc/openvpn/easy-rsa/pki
```
```bash
openvpn_server_ip: "192.168.0.101"
openvpn_port: 1194
openvpn_proto: udp
openvpn_dev: tun
openvpn_user: nobody
openvpn_group: nogroup
openvpn_server_subnet: "10.0.0.0 255.255.255.0"
openvpn_ca: "/etc/openvpn/ca.crt"
openvpn_cert: "/etc/openvpn/server.crt"
openvpn_key: "/etc/openvpn/server.key"
openvpn_dh: "/etc/openvpn/dh.pem"
openvpn_status_log: "/var/log/openvpn/openvpn-status.log"
openvpn_log: "/var/log/openvpn/openvpn.log"
openvpn_persist_key: true
openvpn_persist_tun: true
openvpn_keepalive: "10 120"
```

tasks/main.yml:
```bash
---
- name: Update apt cache
  apt:
    update_cache: yes
  become: yes
  changed_when: false

- name: Install OpenVPN and Easy-RSA
  apt:
    name: "{{ openvpn_packages }}"
    state: present
  become: yes

- name: Ensure Easy-RSA directory exists
  block:
    - name: Check if Easy-RSA directory exists
      stat:
        path: "{{ easy_rsa_path }}"
      register: easy_rsa_exists

    - name: Create Easy-RSA directory if it doesn't exist
      command: make-cadir "{{ easy_rsa_path }}"
      become: yes
      when: not easy_rsa_exists.stat.exists

- name: Ensure PKI directory exists
  block:
    - name: Check if PKI directory exists
      stat:
        path: "{{ easy_rsa_pki_path }}"
      register: pki_exists

    - name: Create PKI directory if it doesn't exist
      command: ./easyrsa init-pki
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not pki_exists.stat.exists

- name: Create and manage CA and server certificates
  block:
    - name: Check if CA certificate already exists
      stat:
        path: "{{ easy_rsa_pki_path }}/ca.crt"
      register: ca_cert

    - name: Build CA without password and common name
      shell: echo "MyOpenVPN_CA" | ./easyrsa build-ca nopass
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not ca_cert.stat.exists

    - name: Check if server key already exists
      stat:
        path: "{{ easy_rsa_pki_path }}/private/server.key"
      register: server_key

    - name: Generate server request without password
      shell: echo "MyOpenVPN_Server" | ./easyrsa gen-req server nopass
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not server_key.stat.exists

    - name: Check if server certificate already exists
      stat:
        path: "{{ easy_rsa_pki_path }}/issued/server.crt"
      register: server_cert

    - name: Request sign for server certificate
      shell: echo "yes" | ./easyrsa sign-req server server
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not server_cert.stat.exists

    - name: Check if Diffie-Hellman parameters already exist
      stat:
        path: "{{ easy_rsa_pki_path }}/dh.pem"
      register: dh_params

    - name: Generate Diffie-Hellman parameters
      command: ./easyrsa gen-dh
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not dh_params.stat.exists

    - name: Check if TLS key already exists
      stat:
        path: "{{ easy_rsa_pki_path }}/ta.key"
      register: tls_key

    - name: Generate TLS key
      command: openvpn --genkey --secret pki/ta.key
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not tls_key.stat.exists

- name: Ensure OpenVPN directory
  file:
    path: /etc/openvpn
    state: directory
    mode: '0755'
  become: yes

- name: Copy required files to OpenVPN directory
  copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    remote_src: yes
    force: no
  with_items:
    - { src: "{{ easy_rsa_pki_path }}/ca.crt", dest: "{{ openvpn_config_path }}/ca.crt" }
    - { src: "{{ easy_rsa_pki_path }}/issued/server.crt", dest: "{{ openvpn_config_path }}/server.crt" }
    - { src: "{{ easy_rsa_pki_path }}/private/server.key", dest: "{{ openvpn_config_path }}/server.key" }
    - { src: "{{ easy_rsa_pki_path }}/dh.pem", dest: "{{ openvpn_config_path }}/dh.pem" }
    - { src: "{{ easy_rsa_pki_path }}/ta.key", dest: "{{ openvpn_config_path }}/ta.key" }
  become: yes

- name: Apply OpenVPN server configuration
  template:
    src: server.conf.j2
    dest: /etc/openvpn/server.conf
  become: yes

#- name: Start OpenVPN server
#  systemd:
#    name: openvpn@server
#    state: started
#    enabled: yes

- name: Check if OpenVPN is running
  shell: pgrep -f "openvpn --config /etc/openvpn/server.conf"
  register: openvpn_process
  ignore_errors: true
  changed_when: false

- name: Start OpenVPN server manually
  command: openvpn --config /etc/openvpn/server.conf
  async: 30
  poll: 0
  become: yes
  when: openvpn_process.rc != 0

- name: Create client configuration file
  template:
    src: client.ovpn.j2
    dest: "client.ovpn"
```

Добавим шаблон для сервера, на котором мы разворачиваем роль openvpn в файле /openvpn/templetes/server.conf.j2:

```bash
port {{ openvpn_port }}
proto {{ openvpn_proto }}
dev {{ openvpn_dev }}
server {{ openvpn_server_subnet }}
ifconfig-pool-persist ipp.txt
push "route {{ openvpn_server_subnet }}"
user {{ openvpn_user }}
group {{ openvpn_group }}
{% if openvpn_persist_key %}
ca {{ openvpn_ca }}
cert {{ openvpn_cert }}
key {{ openvpn_key }}
dh {{ openvpn_dh }}
log-append {{ openvpn_log }}
persist-key
{% endif %}
{% if openvpn_persist_tun %}
persist-tun
{% endif %}
keepalive {{ openvpn_keepalive }}
status {{ openvpn_status_log }} 1
```

Также добавим шаблон для клиент-сервера в файле /openvpn/templetes/client.ovpn.j2:

```bash
client
proto {{ openvpn_proto }}
dev {{ openvpn_dev }}
remote {{ openvpn_server_subnet }}
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
cipher AES-256-CBC
```

Изменим конфигурационный файл molecule.yml для тестрования ansible роли:

```bash
---
dependency:
  name: galaxy
  options:
    ignore-certs: True
    ignore-errors: True
    role-file: requirements.yml

driver:
  name: docker

platforms:
  - name: instance
    image: geerlingguy/docker-ubuntu2004-ansible
    privileged: true
    pre_build_image: true

provisioner:
  name: ansible

verifier:
  name: ansible
```
пушим все на гит
<img width="512" alt="image" src="https://github.com/user-attachments/assets/8bab4c80-5221-4c36-a1dd-1eceff5be24f" />

