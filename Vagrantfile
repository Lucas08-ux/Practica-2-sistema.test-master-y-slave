# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  config.vm.provision "shell", inline: <<-SHELL
     apt-get update
     apt-get install -y apache2
  SHELL

  config.vm.define "venus" do |venus|
    venus.vm.box = "debian/bullseye64"
    venus.vm.network "private_network", ip: "192.168.57.102"

    venus.vm.hostname = "venus.sistema.test"
  end # venus

  config.vm.define "tierra" do |tierra|
    tierra.vm.box = "debian/bullseye64"
    tierra.vm.network "private_network", ip: "192.168.57.103"

    tierra.vm.hostname = "tierra.sistema.test"
  end # tierra

end
