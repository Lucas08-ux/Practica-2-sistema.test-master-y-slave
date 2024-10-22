"# Practica-2-sistema.test-master-y-slave" 

# He creado primero los archivos necesarios, que son .gitignore, Vagrantfile, README.md y LICENSE.

# En .gitignore he escrito esto para que ignore el directorio .vagrant y los backup con las extensiones más comunes:
```
.vagrant/

*~
*.bat
*.bak
*.tmp
*.swp
```
# He modificado Vagrantfile, añadiendo dos máquinas virtuales, que son venus y tierra:
```
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
```
# He escrito "vagrant up" para que se instalen las MV y se actualicen.

# He añadido estas línea a /etc/bind/named.conf.options para que venus y tierra solo hagan escucha en protocolo IPv4:
```
  listen-on-v6 { none; };
```
# He añadido a Vagrantfile esta línea para que se instale bind9 (también se puede instalar de manera manual)
```
  apt-get install -y bind9
```
# He modificado el archivo /etc/bind/named.conf.options en venus y tierra, poniendo dnssec-validation yes
```
sudo nano /etc/bind/named.conf.options
```
# Ya en el fichero

```
options {
    directory "/var/cache/bind";

    dnssec-validation yes;  # Aquí se añade la línea

    listen-on-v6 { none; };
};
```

# Edito de nuevo /etc/bind/named.conf.options en venus y tierra y le añado esto:
```
acl "permitted" {
    127.0.0.0/8;           // Permitir consultas desde localhost
    192.168.57.0/24;       // Permitir consultas desde la red 192.168.57.0/24
};
```
# En el mismo archivo, dentro del bloque options, le añado esta línea para permitir la recursividad
```
allow-recursion { permitted; };  // Permitir la recursión solo a la ACL definida
```
# Así queda al final:
```
options {
	directory "/var/cache/bind";

	allow-recursion { permitted; };

	dnssec-validation yes;

	listen-on-v6 { none; };
};

acl "permitted" {
	127.0.0.0/8;
	192.168.57.0/24;
};
```

# 