Vagrant.configure("2") do |config|
  # Base VM OS configuration.
  config.vm.box = "centos/7"
  config.vm.box_version = "2004.01"
  config.vm.provider :virtualbox do |v|
    v.memory = 512
    v.cpus = 1
  end
  # Define two VMs with static private IP addresses.
  boxes = [
    { :name => "backup",
      :ip => "192.168.11.160",
    },
    { :name => "client",
      :ip => "192.168.11.150",
    },
    { :name => "master",
      :ip => "192.168.11.140",
    }

  ]
  #Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network "private_network", ip: opts[:ip]
      if opts[:name] == boxes.last[:name]
#        config.vm.provision "ansible" do |ansible|
#          ansible.playbook = "ansible/provision.yml"
#          ansible.inventory_path = "ansible/hosts"
#          ansible.host_key_checking = "false"
#          ansible.limit = "all"
#          ansible.verbose = "true"
#        end
      end
    end
  end
end

