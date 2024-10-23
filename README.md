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

# Modifico named.conf.local de tierra y le pongo reverse

```
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "tierra.sistema.test" {
	type master;
	file "/var/lib/bind/tierra.sistema.test.dns";
	allow-transfer { 192.168.57.102; };
};

//Inversa

zone "57.168.192.in-addr.arpa" {
	type master;
	file "/var/lib/bind/tierra.sistema.test.rev";
	allow-transfer { 192.168.57.102; };	
};
```

# Zona directa maestro
```
;
; tierra.sistema.test
;

$TTL	86400
@ IN SOA debian.tierra.sistema.test. admin.tierra.sistema.test. (
	1	; Serial
	3600	; Refresh
	1800	; Retry
	604800	; Expire
	86400 )	; Negative Cache TTL
;
@ IN NS debian.tierra.sistema.test.
debian.tierra.sistema.test. IN A 192.168.57.103
```
# Zona inversa maestro
```
;
; 103.57.168.192
;

$TTL	86400
@ IN SOA debian.tierra.sistema.test. admin.tierra.sistema.test. (
	1	; Serial
	3600	; Refresh
	1800	; Retry
	604800	; Expire
	86400 )	; Negative Cache TTL
;
@ IN NS debian.tierra.sistema.test.
103 IN PTR debian.tierra.sistema.test.
```

# Modifico named.conf.local de venus y le pongo reverse
```
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "tierra.sistema.test" {
    type slave;
    file "/var/lib/bind/tierra.sistema.test.zone";
    masters { 192.168.57.103; }; # IP del servidor maestro (tierra)
};

zone "57.168.192.in-addr.arpa" {
    type slave;
    file "/var/lib/bind/tierra.sistema.test.rev";
    masters { 192.168.57.103; }; # IP del servidor maestro (tierra)
};
```
# He puesto esta línea en los archivos de zona directa e inversa para que el tiempo caché de las respuestas sea de 2 horas
```
7200     ; Negative Cache TTL (2 horas en segundos)
```
# He modificado el archivo named.conf.options para que consultas que reciba el servidor para la que no está autorizado, deberá reenviarlas (forward) al servidor DNS 208.67.222.222 (OpenDNS)
```
//Configuro reenvio
	forward only;
	forwarders {
		208.67.222.222;
	};
```
# He modificado los dos archivos de zonas directa e inversa, añadiendo también los dos servidores imaginarios marte y mercurio.
```
;
; tierra.sistema.test
;

$TTL	86400
@ IN SOA debian.tierra.sistema.test. admin.tierra.sistema.test. (
	1	; Serial
	3600	; Refresh
	1800	; Retry
	604800	; Expire
	7200 )	; Negative Cache TTL
;
@ IN NS debian.tierra.sistema.test.

@ IN A 192.168.57.103

mercurio IN A 192.168.57.101
marte IN A 192.168.57.104
venus IN A 192.168.57.102
debian IN A 192.168.57.103

; Alias
ns1 IN CNAME tierra.sistema.test.  ; ns1 es un alias de tierra
ns2 IN CNAME venus.sistema.test.    ; ns2 es un alias de venus
mail IN CNAME marte.sistema.test.    ; mail es un alias de marte
```
# Y esta es la inversa
```
;
; 103.57.168.192
;

$TTL	86400
@ IN SOA debian.tierra.sistema.test. admin.tierra.sistema.test. (
	1	; Serial
	3600	; Refresh
	1800	; Retry
	604800	; Expire
	7200 )	; Negative Cache TTL
;
@ IN NS debian.tierra.sistema.test.
103 IN PTR debian.tierra.sistema.test.
101 IN PTR mercurio.sistema.test.
102 IN PTR venus.sistema.test.
103 IN PTR tierra.sistema.test.
104 IN PTR marte.sistema.test.
```

# He añadido el correo de marte en tierra.sistema.test.dns
```
@ IN MX 10 marte.sistema.test.
```

# He realizado varias pruebas con dig A. Aquí muestro dos de los diversos resultados que he tenido. Todos funcionando correctamente:
```
vagrant@tierra:~$ dig A marte.sistema.test

; <<>> DiG 9.18.28-1~deb12u2-Debian <<>> A marte.sistema.test
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 31798
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;marte.sistema.test.            IN      A

;; Query time: 3 msec
;; SERVER: 10.0.2.3#53(10.0.2.3) (UDP)
;; WHEN: Wed Oct 23 16:31:41 UTC 2024
;; MSG SIZE  rcvd: 36

vagrant@tierra:~$ dig A tierra.sistema.test

; <<>> DiG 9.18.28-1~deb12u2-Debian <<>> A tierra.sistema.test
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 41199
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;tierra.sistema.test.           IN      A

;; Query time: 3 msec
;; SERVER: 10.0.2.3#53(10.0.2.3) (UDP)
;; WHEN: Wed Oct 23 16:31:52 UTC 2024
;; MSG SIZE  rcvd: 37
```

# Compruebo que se pueden resolver de forma inversa sus direcciones IP
