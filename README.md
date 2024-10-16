"# Practica-2-sistema.test-master-y-slave" 

He creado primero los archivos necesarios, que son .gitignore, Vagrantfile,
README.md y LICENSE.

En .gitignore he escrito esto para que ignore el directorio .vagrant y los 
backup con las extensiones más comunes:

# Ignorar el directorio .vagrant
.vagrant/

# Ignorar ficheros de backup comunes
*~
*.bat
*.bak
*.tmp
*.swp


He modificado Vagrantfile, añadiendo dos máquinas virtuales, que son venus y tierra:

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

He escrito "vagrant up" para que se instalen las MV y 
se actualicen.

