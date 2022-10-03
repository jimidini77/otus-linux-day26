# -*- mode: ruby -*-
# vim: set ft=ruby :

Vagrant.configure("2") do |config|
config.vm.synced_folder ".", "/vagrant", disabled: true
file_to_disk = './sata1.vdi'

  config.vm.define "backup" do |backup|
    backup.vm.box = "centos/7"
    backup.vm.network "private_network", ip: "192.168.56.160"
    backup.vm.hostname = "backup"
    backup.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
      needsController = false
      unless File.exist?(file_to_disk)
        vb.customize ['createhd', '--filename', file_to_disk, '--variant', 'Fixed', '--size', 2048]
        needsController =  true
      end
      if needsController == true
        vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
        vb.customize ['storageattach', :id, '--storagectl', 'SATA', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', file_to_disk]
     end
    end
  end

  config.vm.define "client" do |client|
    client.vm.box = "centos/7"
    client.vm.network "private_network", ip: "192.168.56.150"
    client.vm.hostname = "client"
    client.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
    end
    client.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/provision.yml"
      ansible.inventory_path = "ansible/hosts"
      ansible.host_key_checking = "false"
      ansible.limit = "all"
      ansible.verbose = "true"
    end
  end


  config.vm.define "master" do |master|
    master.vm.box = "centos/7"
    master.vm.network "private_network", ip: "192.168.11.140"
    master.vm.hostname = "master"
    master.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
    end
  end

end