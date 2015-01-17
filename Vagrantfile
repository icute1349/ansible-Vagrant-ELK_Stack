# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  config.vm.box = "mjp182/CentOS_7"

  #config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "private_network", ip: "192.168.33.10"

  # config.vm.synced_folder "../data", "/vagrant_data"
  
  config.vm.provider "virtualbox" do |vb|
     vb.memory = "2046"
     vb.cpus = 4
   end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
#    ansible.start_at_task =""
    ansible.sudo = true
  end
end