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
	 apt-get install -y bind9
  SHELL

  config.vm.define "venus" do |venus|
    venus.vm.box = "debian/bookworm64"
    venus.vm.network "private_network", ip: "192.168.57.102"

    venus.vm.hostname = "venus.sistema.test"
  end # venus

  config.vm.define "tierra" do |tierra|
    tierra.vm.box = "debian/bookworm64"
    tierra.vm.network "private_network", ip: "192.168.57.103"

    tierra.vm.hostname = "tierra.sistema.test"
  end # tierra

end
```
# He añadido estas línea a /etc/bind/named.conf.options para que venus y tierra solo hagan escucha en protocolo IPv4:
```
  listen-on-v6 { none; };
```
# También en el fichero /etc/default named he modificado la línea OPTIONS para que funcione solo en IPv4:
```
#
# run resolvconf?
RESOLVCONF=no

# startup options for the server
OPTIONS="-u bind -4"
```
# He modificado el archivo /etc/bind/named.conf.options en venus y tierra, poniendo dnssec-validation yes
```
sudo nano /etc/bind/named.conf.options
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

zone "sistema.test" {
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
@ IN SOA tierra.sistema.test. admin.tierra.sistema.test. (
	2024102301	; Serial
	3600	; Refresh
	1800	; Retry
	604800	; Expire
	7200 )	; Negative Cache TTL
;
@ IN NS tierra.sistema.test.
@ IN NS venus.sistema.test.

@ IN A 192.168.57.103

mercurio IN A 192.168.57.101
marte IN A 192.168.57.104
venus IN A 192.168.57.102
tierra IN A 192.168.57.103

; Alias
ns1 IN CNAME tierra.sistema.test.  ; ns1 es un alias de tierra
ns2 IN CNAME venus.sistema.test.    ; ns2 es un alias de venus
mail IN CNAME marte.sistema.test.    ; mail es un alias de marte

@ IN MX 10 marte.sistema.test.
```
# Zona inversa maestro
```
;
; 103.57.168.192
;

$TTL	86400
@ IN SOA tierra.sistema.test. admin.tierra.sistema.test. (
	1	; Serial
	3600	; Refresh
	1800	; Retry
	604800	; Expire
	7200 )	; Negative Cache TTL
;
@ IN NS tierra.sistema.test.
103 IN PTR tierra.sistema.test.
101 IN PTR mercurio.sistema.test.
102 IN PTR venus.sistema.test.
104 IN PTR marte.sistema.test.
```

# Modifico named.conf.local de venus y le pongo reverse también
```
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "sistema.test" {
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
# He modificado el archivo named.conf.options para que consultas que reciba el servidor para la que no está autorizado, deberá reenviarlas (forward) al servidor DNS 208.67.222.222 (OpenDNS). Aquí se ve el archivo completo:
```
options {
	directory "/var/cache/bind";

	allow-recursion { permitted; };

	dnssec-validation yes;

	listen-on port 53 { 192.168.57.103; };
	listen-on-v6 { none; };

	//Configuro reenvio
	forward only;
	forwarders {
		208.67.222.222;
	};
};

acl "permitted" {
	127.0.0.0/8;
	192.168.57.0/24;
};
```

# He realizado varias pruebas con dig A. Aquí muestro dos de los diversos resultados que he tenido. Todos funcionando correctamente:
```
vagrant@tierra:~$ dig @192.168.57.103 A tierra.sistema.test

; <<>> DiG 9.18.28-1~deb12u2-Debian <<>> @192.168.57.103 A tierra.sistema.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55789
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: da0ff394882cd7e60100000067194ac601f68dfb972b8bf4 (good)
;; QUESTION SECTION:
;tierra.sistema.test.           IN      A

;; ANSWER SECTION:
tierra.sistema.test.    86400   IN      A       192.168.57.103

;; Query time: 3 msec
;; SERVER: 192.168.57.103#53(192.168.57.103) (UDP)
;; WHEN: Wed Oct 23 19:13:10 UTC 2024
;; MSG SIZE  rcvd: 92

vagrant@tierra:~$ dig @192.168.57.103 A mercurio.sistema.test

; <<>> DiG 9.18.28-1~deb12u2-Debian <<>> @192.168.57.103 A mercurio.sistema.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63709
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: f4647117baf3ffe50100000067194ac65dadd1741a392ba1 (good)
;; QUESTION SECTION:
;mercurio.sistema.test.         IN      A

;; ANSWER SECTION:
mercurio.sistema.test.  86400   IN      A       192.168.57.101

;; Query time: 3 msec
;; SERVER: 192.168.57.103#53(192.168.57.103) (UDP)
;; WHEN: Wed Oct 23 19:13:10 UTC 2024
;; MSG SIZE  rcvd: 94
```

# Compruebo que se pueden resolver de forma inversa sus direcciones IP. Aquí muestro un ejemplo con dig:
```
vagrant@tierra:~$ dig @192.168.57.102

; <<>> DiG 9.18.28-1~deb12u2-Debian <<>> @192.168.57.102
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43848
;; flags: qr rd ra; QUERY: 1, ANSWER: 13, AUTHORITY: 0, ADDITIONAL: 27

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: b2921106844d848601000000671940ccd1b4ce08cbab5ccc (good)
;; QUESTION SECTION:
;.                              IN      NS

;; ANSWER SECTION:
.                       516325  IN      NS      h.root-servers.net.
.                       516325  IN      NS      j.root-servers.net.
.                       516325  IN      NS      d.root-servers.net.
.                       516325  IN      NS      b.root-servers.net.
.                       516325  IN      NS      c.root-servers.net.
.                       516325  IN      NS      f.root-servers.net.
.                       516325  IN      NS      e.root-servers.net.
.                       516325  IN      NS      g.root-servers.net.
.                       516325  IN      NS      a.root-servers.net.
.                       516325  IN      NS      i.root-servers.net.
.                       516325  IN      NS      m.root-servers.net.
.                       516325  IN      NS      l.root-servers.net.
.                       516325  IN      NS      k.root-servers.net.

;; ADDITIONAL SECTION:
a.root-servers.net.     516325  IN      A       198.41.0.4
b.root-servers.net.     516325  IN      A       170.247.170.2
c.root-servers.net.     516325  IN      A       192.33.4.12
d.root-servers.net.     516325  IN      A       199.7.91.13
e.root-servers.net.     516325  IN      A       192.203.230.10
f.root-servers.net.     516325  IN      A       192.5.5.241
g.root-servers.net.     516325  IN      A       192.112.36.4
h.root-servers.net.     516325  IN      A       198.97.190.53
i.root-servers.net.     516325  IN      A       192.36.148.17
j.root-servers.net.     516325  IN      A       192.58.128.30
k.root-servers.net.     516325  IN      A       193.0.14.129
l.root-servers.net.     516325  IN      A       199.7.83.42
m.root-servers.net.     516325  IN      A       202.12.27.33
a.root-servers.net.     516325  IN      AAAA    2001:503:ba3e::2:30
b.root-servers.net.     516325  IN      AAAA    2801:1b8:10::b
c.root-servers.net.     516325  IN      AAAA    2001:500:2::c
d.root-servers.net.     516325  IN      AAAA    2001:500:2d::d
e.root-servers.net.     516325  IN      AAAA    2001:500:a8::e
f.root-servers.net.     516325  IN      AAAA    2001:500:2f::f
g.root-servers.net.     516325  IN      AAAA    2001:500:12::d0d
h.root-servers.net.     516325  IN      AAAA    2001:500:1::53
i.root-servers.net.     516325  IN      AAAA    2001:7fe::53
j.root-servers.net.     516325  IN      AAAA    2001:503:c27::2:30
k.root-servers.net.     516325  IN      AAAA    2001:7fd::1
l.root-servers.net.     516325  IN      AAAA    2001:500:9f::42
m.root-servers.net.     516325  IN      AAAA    2001:dc3::35

;; Query time: 0 msec
;; SERVER: 192.168.57.102#53(192.168.57.102) (UDP)
;; WHEN: Wed Oct 23 18:30:36 UTC 2024
;; MSG SIZE  rcvd: 851
```

# Prueba con los alias ns1 y ns2
```
vagrant@tierra:~$ dig @192.168.57.103 CNAME ns1.sistema.test

; <<>> DiG 9.18.28-1~deb12u2-Debian <<>> @192.168.57.103 CNAME ns1.sistema.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58945
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: c0686b80f0ea6e9b0100000067194a83643efa3ac00effc6 (good)
;; QUESTION SECTION:
;ns1.sistema.test.              IN      CNAME

;; ANSWER SECTION:
ns1.sistema.test.       86400   IN      CNAME   tierra.sistema.test.

;; Query time: 0 msec
;; SERVER: 192.168.57.103#53(192.168.57.103) (UDP)
;; WHEN: Wed Oct 23 19:12:03 UTC 2024
;; MSG SIZE  rcvd: 94

vagrant@tierra:~$ dig @192.168.57.103 CNAME ns2.sistema.test

; <<>> DiG 9.18.28-1~deb12u2-Debian <<>> @192.168.57.103 CNAME ns2.sistema.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15625
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 455cd9b545cac6a10100000067194a8cfac278659b418a4c (good)
;; QUESTION SECTION:
;ns2.sistema.test.              IN      CNAME

;; ANSWER SECTION:
ns2.sistema.test.       86400   IN      CNAME   venus.sistema.test.

;; Query time: 0 msec
;; SERVER: 192.168.57.103#53(192.168.57.103) (UDP)
;; WHEN: Wed Oct 23 19:12:12 UTC 2024
;; MSG SIZE  rcvd: 93
```
# Realizo la consulta para saber los servidores NS de sistema.test.
```
vagrant@tierra:~$ dig @192.168.57.103 NS sistema.test

; <<>> DiG 9.18.28-1~deb12u2-Debian <<>> @192.168.57.103 NS sistema.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5560
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 1869a5954daedcb90100000067194b4f7526a1af446f967b (good)
;; QUESTION SECTION:
;sistema.test.                  IN      NS

;; ANSWER SECTION:
sistema.test.           86400   IN      NS      tierra.sistema.test.
sistema.test.           86400   IN      NS      venus.sistema.test.

;; ADDITIONAL SECTION:
venus.sistema.test.     86400   IN      A       192.168.57.102
tierra.sistema.test.    86400   IN      A       192.168.57.103

;; Query time: 0 msec
;; SERVER: 192.168.57.103#53(192.168.57.103) (UDP)
;; WHEN: Wed Oct 23 19:15:27 UTC 2024
;; MSG SIZE  rcvd: 142
```
#
