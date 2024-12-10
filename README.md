# P9 Apache e VirtualHost

Nesta práctica configurei duás páxinas webs, fabulasmaravillosas e fabulas oscuras, a unha mesma dirección ip do meu DNS, onde os arquivos son os seguintes:

## Docker-compose.yml
Neste arquivo definimos dous servizos: un para as páxinas web e outro para o servidor DNS. Tamén configuramos a rede.
### Web

No apartado web, definimos a imaxe (neste caso, php), o nome do servidor, o porto (que será o 80), os volumes que configuraremos máis adiante, e a rede (que chamei apared).
```
web:
    image: php:7.4-apache
    container_name: apaserver
    ports:
      - "80:80"
    volumes:
      - ./www:/var/www
      - ./confApache:/etc/apache2
    networks:
      apared:
        ipv4_address: 172.39.4.2
```

### DNS

Neste apartado, configuramos a imaxe, os volumes que tamén configuraremos máis adiante, o porto (51 para o servidor DNS), e a rede, que debe ser a mesma pero cunha IP distinta.

```
dns:
    container_name: dnsserver
    image: ubuntu/bind9
    ports:
      - "51:53"
    volumes:
      - ./confDNS/conf:/etc/bind
      - ./confDNS/zonas:/var/lib/bind
    networks:
      apared:
        ipv4_address: 172.39.4.3
```
### Red

Definimos a rede que usaremos, chamada apachered, en modo bridge.

```
networks:
  apachered:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.39.0.0/16
```

## confApache

A carpeta confApache pódese descargar da axuda na aula virtual. Só teremos que engadir dous ficheiros nas carpetas sites-available e sites-enabled. Os ficheiros nas dúas carpetas deben ser os mesmos. Se desexamos usar varios portos, engadimos a liña `listen 8000` no ficheiro ports.conf. Os ficheiros que engadimos son os seguintes:
### fabulasmaravillosas.conf

Neste arquivo configuramos o servidor para o sitio "fabulasmaravillosas", indicando o porto 80, o correo do administrador, o nome do servidor e o alias, así como a ruta onde se atopa o ficheiro index.html.

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost  #admin 
    ServerName fabulasmaravillosas.asircastelao.int  #nome da paxina
    ServerAlias www.fabulasmaravillosas.asircastelao.int #alias da paxina
    DocumentRoot /var/www/fabulasmaravillosas #ruta onde temos o documentoo index.html
</VirtualHost>
```

### fabulasoscuras.conf

Neste arquivo configuramos o servidor para o sitio "fabulasoscuras", de maneira similar ao anterior, pero adaptado a esta páxina web.

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost #admin 
    ServerName fabulasoscuras.asircastelao.int #nome da paxina
    ServerAlias www.fabulasoscuras.asircastelao.int #alias da paxina 
    DocumentRoot /var/www/fabulasoscuras #ruta onde temos o documento index.html
</VirtualHost>
```

## confDNS

Nesta carpeta debemos configurar dúas cousas: a configuración e as zonas. No meu caso empreguei unha única zona.

### zonas

O ficheiro db.nome.int contén a configuración da zona. Neste caso, o nome é asircastelao. Definimos o servidor de nomes principal ns.asircastelao.int e un correo de contacto, e tamén asociamos os nomes das páxinas web ás respectivas IPs.



```
$TTL    604800  
@       IN      SOA     ns.asircastelao.int. some.email.address.com. ( ; Indica o servidor de nomes principal e un correo de contacto.
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative 

@       IN      NS      ns.asircastelao.int. #nome autoritativo
ns 	    IN 	    A 	    172.39.4.3  #asociamos IP ao nome do servidor
fabulasoscuras       IN      A       172.39.4.2  #páxina asociada a IP
fabulasmaravillosas     IN      A       172.39.4.2  # páxina asociada a IP

```

### conf

Nesta carpeta debe haber polo menos tres ficheiros. O primeiro é named.conf.local, onde definimos a zona e indicamos o tipo e a ruta do arquivo da zona:

```
zone "asircastelao.int" {
    type master;
    file "/var/lib/bind/db.asircastelao.int";
    allow-query {
    	any;
    	};
};
```

O seguinte ficheiro é named.conf.options, onde especificamos o directorio, os forwarders (como os de Google), e outras opcións como a resolución recursiva e a permisividade de consultas.

```
options {
    directory "/var/cache/bind";
    recursion yes;                      # Permitir la resolución recursiva
    allow-query { any; };               # Permitir consultas desde cualquier IP
    dnssec-validation no;
    forwarders {
        8.8.8.8;                        # Google DNS
        1.1.1.1;                        # Cloudflare DNS
    };
    listen-on { any; };                 # Escuchar en todas las interfaces
    listen-on-v6 { any; };
};
```
O último ficheiro é named.conf, onde simplemente enlazamos os dous ficheiros anteriores. Tamén poderiamos incluílos directamente neste ficheiro.

```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```

Tamén teño un ficheiro chamado named.conf.default-zones, onde configuro as zonas inversas.



```
// prime the server with knowledge of the root servers
zone "." {
	type hint;
	file "/usr/share/dns/root.hints";
};

// be authoritative for the localhost forward and reverse zones, and for
// broadcast zones as per RFC 1912

zone "localhost" {
	type master;
	file "/etc/bind/db.local";
};

zone "127.in-addr.arpa" {
	type master;
	file "/etc/bind/db.127";
};

zone "0.in-addr.arpa" {
	type master;
	file "/etc/bind/db.0";
};

zone "255.in-addr.arpa" {
	type master;
	file "/etc/bind/db.255";
};
```

## www

Na carpeta www teremos dúas subcarpetas, chamadas fabulasmaravillosas e fabulasoscuras. Dentro de cada unha delas, teremos o ficheiro index.html correspondente a cada páxina web. Como se observou no ficheiro docker-compose.yml, empregamos a ruta /var/www/ para enlazar os index.html, e para iso usaremos os seguintes comandos:
```
sudo mkdir /var/www/fabulasmaravillosas
sudo nano /var/www/fabulasmaravillosas/index.html
```

Os mesmos pasos deben repetirse para a páxina fabulasoscuras.

Se ao crear as carpetas non nos deixa, pode ser porque apache2 non está instalado. Para isto, usamos o comando sudo apt install apache2, cambiamos a configuración do firewall con sudo ufw allow 'Apache', e reiniciamos co comando sudo systemctl restart apache2.

## configuracions extras

Ademais dos pasos anteriores, debemos realizar algúns cambios na nosa máquina. Primeiro, debemos editar o ficheiro /etc/systemd/resolved.conf co comando sudo nano /etc/systemd/resolved.conf e descomentar a liña do DNS, engadindo a IP do noso servidor DNS e o porto correspondente:

```
DNS=172.39.4.3#51
```

Despois, gardamos os cambios co comando sudo systemctl restart systemd-resolved.

Finalmente, debemos acceder á configuración de rede da nosa máquina, desactivar a opción de DNS automático na sección de IPv4, e reiniciar o equipo.

## comprobación

Unha vez feitos todos os pasos, lanzamos o comando docker compose up e, se todo está correctamente configurado, podemos acceder ás nosas páxinas web no navegador, por exemplo, en www.fabulasmaravillosas.asircastelao.int ou www.fabulasoscuras.asircastelao.int, e ver o contido dos ficheiros index.html.

Se ao lanzar docker compose up aparece un erro, pode ser porque apache2 xa está en funcionamento, así que debemos paralo co comando sudo systemctl stop apache2.

