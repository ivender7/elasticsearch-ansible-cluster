# -*- mode: ruby -*-
# vi: set ft=ruby :

NODES = {
  "es-node-1" => "192.168.56.11",
  "es-node-2" => "192.168.56.12",
  "es-node-3" => "192.168.56.13"
}

NODE_MEMORY = 2048
NODE_CPUS   = 2
BOX         = "bento/ubuntu-24.04"

Vagrant.configure("2") do |config|
  config.vm.box = BOX
  config.vm.box_check_update = false

  NODES.each do |name, ip|
    config.vm.define name do |node|
      node.vm.hostname = name
      node.vm.network "private_network", ip: ip

      node.vm.provider "virtualbox" do |vb|
        vb.name   = name
        vb.memory = NODE_MEMORY
        vb.cpus   = NODE_CPUS
        vb.customize ["modifyvm", :id, "--audio", "none"]
        vb.customize ["modifyvm", :id, "--usb", "off"]
      end

      node.vm.provision "shell", inline: <<-SHELL
        set -e
      SHELL
    end
  end
end
