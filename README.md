# Лабораторная Работа №1
## 1. Установка и настройка Vagrant
Работа выполнялось на Windows. Vagrant был установлен, скачиванием установочного файла с официального сайта. Для дальнейшей настройки Vagrant, понадобится VirtualBox дя разварачивания машин для ЛР

Для настройки Vagrant проинициализируем его командой:
```bash
vagrant init
```

Данное действие создаст Vagrantfile, а также папку .vagrant.d, в которой в папку boxes, мы поместим образ ОС, которая в будущем будет стоять на наших nodes

![image](https://github.com/user-attachments/assets/a3c07bfc-7f61-4fa7-893c-ef3127b2ac9a)

Далее откроем созданный Vagrantfile и напишем скрипт для поднятия master / worker машин:
 ```bash
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"

  BRIDGE = "Intel(R) Wi-Fi 6 AX200 160MHz"
    
  (1..4).each do |i|
    config.vm.define "worker#{i}" do |worker|
      worker.vm.network "private_network", ip: "192.168.0.10#{i}"
      worker.vm.network "public_network", bridge: BRIDGE
      worker.vm.hostname = "worker#{i}"
      worker.vm.provider "virtualbox" do |vb|
        vb.memory = 1024
        vb.cpus = 1
      end
      worker.vm.provision "shell", inline: <<-SHELL
        sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
        sudo sed -i 's/^#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config
        sudo systemctl restart sshd
      SHELL
    end
  end
  
  config.vm.define "master" do |master|
    master.vm.network "private_network", ip: "192.168.0.100"
    master.vm.network "public_network", bridge: BRIDGE
    master.vm.hostname = "master"
    master.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
    end
    master.vm.provision "shell", inline: <<-SHELL
      sudo apt update && apt install -y sshpass

      if [ ! -f /home/vagrant/.ssh/id_rsa ]; then
        ssh-keygen -t rsa -b 2048 -f /home/vagrant/.ssh/id_rsa -N ''
      fi

      sudo chmod 444 /home/vagrant/.ssh/id_rsa

      for i in {101..104}
      do
        sshpass -p vagrant ssh-copy-id -o StrictHostKeyChecking=no -i /home/vagrant/.ssh/id_rsa.pub vagrant@192.168.0.$i || echo "Failed to copy key to 192.168.0.$i"
      done
    SHELL
  end
end
```

## Как работает Vagrantfile:
 1. Базовый образ: Используется официальный образ Ubuntu 20.04 (ubuntu/focal64). Этот образ скачивается автоматически при первом запуске.
 2. Private Network: Каждая виртуальная машина получает статический IP-адрес в локальной сети (192.168.0.xxx).
 3. Public Network: Настраивается доступ к интернету через мостовой адаптер (ваш реальный сетевой интерфейс).
 3. Ресурсы: Для каждого сервера указывается объём памяти и количество ядер.
 4. Настройка SSH: Master автоматически передаёт свой публичный ключ SSH всем worker-узлам для упрощённого подключения, для этого пришлось поменять      настройки файла sshd_config

Команда для поднятия машин: 
```bash
vagrant up
```
После поднятия машин, проверяем статус машин:

![image](https://github.com/user-attachments/assets/0f51580c-7bec-4840-b412-33a3ce432f00)

Проверяем доступ к машинам:

![image](https://github.com/user-attachments/assets/a4fbf038-dd92-480a-a2a4-8da1b7a2deb3)

## 2. Ansible и Inventory файл 

Для дальнейшей работы скачиваем нужные библиотеки для Ansible и сам Ansible на master ноде:

![image](https://github.com/user-attachments/assets/6ac53244-bd1c-4b58-943c-0dd5f2e9cff4)
![image](https://github.com/user-attachments/assets/16a7fdc5-8dbe-446e-86c1-bb3559b34f4b)
![image](https://github.com/user-attachments/assets/c4f3ff69-7fcb-4866-8323-88306d1ed0ff)

После успешной установки Ansible, напишем inventory файл:

![image](https://github.com/user-attachments/assets/cff6a83d-05ef-4af0-83a2-1748deec9325)

## 3. Написание playbook для установки Docker

Создадим в директории ansible_project/playbooks/ новый lab1_playbook:

```yaml
---
- hosts: app
  become: true
  tasks:
  - name: Install required system packages
    apt:
        pkg:
        - curl
        - git
        - lynx
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

  - name: Clone github repo
    git:
        repo: 'https://github.com/mdn/django-locallibrary-tutorial.git'
        dest: /home/{{ ansible_user }}/app
        version: main
        update: yes

  - name: Pull the Django Docker image
    docker_image:
        name: timurbabs/django
        tag: latest
        source: pull

  - name: Run the Django container
    docker_container:
        name: django_app
        image: timurbabs/django:latest
        state: started
        restart_policy: always
        ports:
          - "8000:8000"
```

## 4. Запускаем приложение и проверяем работоспособность

Для запуска используем следующую команду:

```bash
ansible-playbook -i inventory.yml -l app -u vagrant lab1_plb.yml
```

![image](https://github.com/user-attachments/assets/b71519b5-2327-4c57-b740-59aa908fac51)
![image](https://github.com/user-attachments/assets/9e4906fd-cb3f-44e2-826b-726775ad57d7)

Проверим работу, зайдя на один из серверов, используя команду:

```bash
lynx http://localhost:8000
```

В конечном итоге увидим следующее:

![image](https://github.com/user-attachments/assets/f513ce86-1790-4288-9470-eae1b4f7a0dc)

