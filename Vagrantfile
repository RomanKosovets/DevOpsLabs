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
