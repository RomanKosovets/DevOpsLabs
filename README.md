# Лабораторная Работа №1
## Установка и настройка Vagrant
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
    
  (1..1).each do |i|
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

      for i in {101..102}
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
Для каждого сервера увидим вот такой результат:

image

image

Попробуем подключиться к одному из серверов:

image

Успешно!
