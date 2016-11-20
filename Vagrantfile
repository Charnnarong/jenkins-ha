# -*- mode: ruby -*-
# vi: set ft=ruby :
required_plugins = %w(vagrant-vbguest vagrant-persistent-storage)
required_plugins.each do |plugin|
  system "vagrant plugin install #{plugin}" unless Vagrant.has_plugin? plugin
end

Vagrant.configure("2") do |config|

  # Docker Node
  config.vm.define "centos-gateway" do |d|
    d.vm.box = "centos/7"
    d.vm.hostname = "centos-gateway"
    d.vm.network "private_network" , ip: "10.100.199.10"
    d.vm.synced_folder "./resources", "/vagrant"
    d.vm.provision :shell, path: "scripts/passwordAuthentication.sh"
    d.vm.provider "virtualbox" do |v|
      v.memory = 2048
    end
  end

  drdb_persistent_storage = 2048
  # Docker Node
  config.vm.define "centos-active" do |d|
    d.vm.box = "centos/7"
    d.vm.hostname = "centos-active"
    d.vm.network "private_network" , ip: "10.100.199.20"
    d.vm.synced_folder "./resources", "/vagrant"
    d.vm.provision :shell, path: "scripts/passwordAuthentication.sh"
=begin 
    # TODO : Set persistent storage and find the way to attatch to drdb. 
    d.persistent_storage.enabled = true
    d.persistent_storage.location = "centosactivevd.vdi"
    d.persistent_storage.size = "#{drdb_persistent_storage}"
    d.persistent_storage.mountname = 'jenkinsdata'
    d.persistent_storage.filesystem = 'ext4'
    d.persistent_storage.mountpoint = '/var/lib/jenkinsdata'
    d.persistent_storage.volgroupname = 'VolGroup00'
=end    
    d.vm.provider "virtualbox" do |v|
      v.memory = 2048
    end
  end
 
   # Docker Node
  config.vm.define "centos-passive" do |d|
    d.vm.box = "centos/7"
    d.vm.hostname = "centos-passive"
    d.vm.network "private_network" , ip: "10.100.199.30"
    d.vm.synced_folder "./resources", "/vagrant"
    d.vm.provision :shell, path: "scripts/passwordAuthentication.sh"
=begin
    d.persistent_storage.enabled = true
    d.persistent_storage.location = "centospassivevd.vdi"
    d.persistent_storage.size = "#{drdb_persistent_storage}"
    d.persistent_storage.mountname = 'jenkinsdata'
    d.persistent_storage.filesystem = 'ext4'
    d.persistent_storage.mountpoint = '/var/lib/jenkinsdata'
    d.persistent_storage.volgroupname = 'VolGroup00'
=end
    d.vm.provider "virtualbox" do |v|
      v.memory = 2048
    end
  end
 

  config.vm.define "centos-ansible" do |d|
    d.vm.box = "centos/7"
    d.vm.hostname = "centos-ansible"
    d.vm.network "private_network" , ip: "10.100.196.10"
    d.vm.synced_folder ".", "/vagrant", rsync__exclude: ".git/"
    d.vm.synced_folder ".vagrant", "/vagrants/pk", mount_options: ["dmode=700,fmode=600"]
    d.vm.provision :shell, path: "scripts/bootstrap_ansible.sh"

    playbooks = [["/vagrant/resources/ansible/provisionHaSystem.yml","/vagrant/resources/ansible/hosts/haSystems.yml"],
                 ["/vagrant/resources/ansible/provisionGateway.yml","/vagrant/resources/ansible/hosts/gateway.yml"]]
    playbooks.each { | (playbook,inventory) |
      d.vm.provision "ansible_local" do |ansible|
        ansible.playbook = "#{playbook}"
        ansible.install = false
        ansible.verbose = true
        ansible.limit          = "all"
        ansible.inventory_path = "#{inventory}"

      end
    }
    d.vbguest.auto_update = true
  end

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
    config.vbguest.no_remote = true
  end

end
