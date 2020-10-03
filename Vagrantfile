# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.box_check_update = true
  config.vm.provider "virtualbox" do |vb|
    vb.customize ['modifyvm', :id, '--nested-hw-virt', 'on']
  end
  config.vm.provision "shell", inline: "sed -i '/#Password.*/s/^#//' /etc/ssh/sshd_config"
  config.vm.provision "shell", inline: "systemctl restart sshd"
  
  config.vm.define "master-k8" do |master|
    master.vm.hostname = "master-k8"
    master.vm.network "private_network", ip: "10.10.10.10"
    master.vm.network "public_network", bridge: "Killer Wireless-n/a/ac 1535 Wireless Network Adapter"
    master.vm.provider "virtualbox" do |v|
      v.name = "master_k8"
      v.memory = 2048
      v.cpus = 2
    end
  end

  $instance=2
  (1..$instance).each do |i|
         config.vm.define "node-k8#{i}" do |node|
           node.vm.hostname =  "node-k8#{i}"
           node.vm.network "private_network", ip: "10.10.10.#{i+10}"
           node.vm.provider "virtualbox" do |v|
            v.name = "node_k8#{i}"
            v.memory = 1024
            v.cpus = 1
           end
         end
  end
end
