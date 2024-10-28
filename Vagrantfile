# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
    :web => {
        :box_name => "bento/ubuntu-22.04",
        :cpus => 1,
        :memory => 1024,
        :ip_addr => '192.168.56.10',
	:playbook => "Ansible/web.yml",
    },
    :log => {
        :box_name => "almalinux/9",
        :cpus => 1,
        :memory => 1024,
        :ip_addr => '192.168.56.13',
        :playbook => "Ansible/log.yml",
    },
}

Vagrant.configure("2") do |config|
  if Vagrant.has_plugin?("vagrant-vbguest")
      config.vbguest.auto_update = false
      config.vbguest.no_remote = true
          end 
  MACHINES.each do |boxname, boxconfig|
      config.vm.define boxname do |box|
          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s
          box.vm.network "private_network", ip: boxconfig[:ip_addr]
          box.vm.provider :virtualbox do |vb|
            vb.name = boxname.to_s
            vb.memory = boxconfig[:memory]
            vb.cpus = boxconfig[:cpus]
          end
	  box.vm.provision "ansible" do |ansible|
            ansible.playbook = boxconfig[:playbook]
      	  end
       end
    end
end



  
