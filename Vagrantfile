# -*- mode: ruby -*-
# vi: set ft=ruby :

PRIVATE_IP="192.168.33.10"
VB_MEM="1024"
VB_CPUS="4"

Vagrant.configure(2) do |config|
  config.vm.box = "mjp182/CentOS_7"

  config.vm.network "private_network", ip: PRIVATE_IP

  # config.vm.synced_folder "../data", "/vagrant_data"
  
  config.vm.provider "virtualbox" do |vb|
     vb.memory = VB_MEM
     vb.cpus = VB_CPUS
   end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
    ansible.sudo = true
  end
end