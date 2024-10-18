"# Practica-2-sistema.test-master-y-slave" 

He creado primero los archivos necesarios, que son .gitignore, Vagrantfile,
README.md y LICENSE.

En .gitignore he escrito esto para que ignore el directorio .vagrant y los 
backup con las extensiones más comunes:

.vagrant/

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

He añadido estas líneas a Vagrantfile para que venus y tierra solo hagan
escucha en protocolo IPv4:

config.vm.provision "shell", inline: <<-SHELL
     apt-get update
     apt-get install -y apache2

     #Modifico la configuración para que Apache escuche solo en IPv4
     sed -i 's/^Listen 80$/Listen 0.0.0.0:80/' /etc/apache2/ports.conf

     #Reinicio Apache para aplicar los cambios
     systemctl restart apache2
  SHELL

Aquí muestro el comando que he usado y el archivo ports.conf una vez se ha modificado

vagrant@tierra:~$ cat /etc/apache2/ports.conf

Listen 0.0.0.0:80

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>
