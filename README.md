# P9 Apache e VirtualHost

Nesta práctica configurei duás páxinas webs, fabulasmaravillosas e fabulas oscuras, a unha mesma dirección ip do meu DNS, onde os arquivos son os seguintes:

## Docker-compose.yml
Neste arquivo temos dous servicios, a das páxinas webs e o do servidor DNS, e ademáis definimos a red.

### Web

Nesta parte temos definido a imaxe, neste caso php, o nome do servidor, o porto que empregaremos, que sera o 80, os volumes que máis adiante configuramos e a red, que a chamei apared.
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

Igual que nas anteriores prácticas, configuramos a imaxe, os volumes que configuraremos máis adiante, o porto que empregaremos e a red, que ten que ser a mesma pero cunha ip distinta.

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

Neste apartado do documento debemos definir a red que imos empregar, a que chamamos apared, en modo bridge.

```
networks:
  apared:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.39.0.0/16
```

## confApache

Esta carpeta podemola descargar da axuda que temos de ti na aula virtual, onde so debemos engadir dous ficheiros na carpeta "sites-available" pero o importante é na carpeta "sites-enable". Este dous ficheiros son os mismos para ambas carpeta. Tamén, no caso de querer empregar varios portos, debemos engadir no ficheiro "ports.conf" a liña `listen 8000`. Os ficheiros que engadiremos as carpetas son os seguintes: 

### fabulasmaravillosas.conf

Neste arquivo debemos chamar ao porto 80 e configurar o seguinte: decir quen vai ser o admin, un correo xeralemnte; o nome e alias da nosa páxina web chamada fabulasmaravillosas, que será o que buscaremos no navegador para ver a nosa páxina; e o ruta onde teremos o noso index.html, que no meu caso teñoa creada nesa ruta.

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost  #admin 
    ServerName fabulasmaravillosas.asircastelao.int  #nome da paxina
    ServerAlias www.fabulasmaravillosas.asircastelao.int #alias da paxina
    DocumentRoot /var/www/fabulasmaravillosas #ruta onde temos o documentoo index.html
</VirtualHost>
```

### fabulasoscuras.conf

Neste arquivo faremos o mesmo que no anterior, só que adaptandoo a esta páxina web, chamada fabulasoscuras.

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost #admin 
    ServerName fabulasoscuras.asircastelao.int #nome da paxina
    ServerAlias www.fabulasoscuras.asircastelao.int #alias da paxina 
    DocumentRoot /var/www/fabulasoscuras #ruta onde temos o documento index.html
</VirtualHost>
```

## confDNS

Nesta carpeta debemos configurar dúass cousas, a configuración, e as zonas, no meu caso empreguei unha única zona.

### zonas

Nesta carpeta debemos que ter o documento **db.nome.int** , que no meu caso o nome é asircastelao. Dentro deste documento, primero debemos empregar o servidor de nomes principal, no meu caso ns.asircastelao.int, e un correo de contacto; declaramos un rexistro NS que indica o servidor de nomes autoritativos para o dominio, neste caso ns.asircastelao.int; asociamos o nome do servidor á dirección IP do dns, no meu caso 172.39.4.3; e asociamos as páxinas webs a dirección IP da nosa web que definimos no docker-compose.yml, neste caso 172.39.4.2



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

Nesta carpeta debemos ter al menos tres documentos. O primerio de todos é o **named.conf.local**, onde debemos chamar ao arquivo da zona, indicando o tipo e a ruta:

```
zone "asircastelao.int" {
    type master;
    file "/var/lib/bind/db.asircastelao.int";
    allow-query {
    	any;
    	};
};
```

O seguinte documento é **named.conf.options** onde primeiro indicamos o directorio, os forwarders como o de google, e outras opcións como a resolucion recursiva, permitir consultas e escoitar todas as interfaces:

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
O último documento é **named.conf** onde simplemente enlazamos os dous documentos anteriores, aunque poderíams facer os dous documentos anteriores neste.

```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```

Ademáis teño outro documento chamado **named.conf.default-zones** onde teño definido as zoas inversas.

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

Nesta carpeta teremos outras dúas carpetas, chamadas exactemente que as páxinas e a ruta que nos documentos de configuración de apache, é decir, no meu caso unha carpeta chamada **fabulasmaravillosas** e outra carpeta chamada **fabulasocuras**. Dentro de cada carpeta, debemos ter os documentos index.html de cada páxina web.

No meu caso, como se observou no documento docker-compose.yml, empreguei a ruta /var/www/ para enlazar os index.html, entonces empreguei os comandos `sudo mkdir /var/www/fabulasmaravillosas` e `sudo nano /var/www/fabulasmaravillosas/index.html`,onde escribín o documento html. Estos mesmos comandos repítense para a páxina fabulasoscuras.

No caso de que á hora de crear as carpetas non te deixa xa que non tes apache descargado, facer os seguintes pasos: instalar apache2 co comando `sudo apt install apache2`,  e cambiar a configuración do firewall cos comandos `sudo ufw allow 'Apache'`, e por ultimo reiniciamos co comando `sudo systemctl restart apache2`

## configuracions extras

Además de todos os pasos anteriores, debemos realizar uns cambios na nosa máquina, o primeiro será entrar nun documento co comando `sudo nano /etc/systemd/resolved.conf` onde debemos de buscar a liña do DNS e descomentala, e temos que añadir a dirección IP do noso DNS que configuramos e o porto que empregamos, no meu caso :

```
DNS=172.39.4.3#51
```

Agora debemos gardar os cambios co comando `sudo systemctl restart systemd-resolved`.

Unha vez feito, debemos entrar na confiugaración de red da nosa máquina, e se vamos ao apartado de IPv4, debemos desactivar a opción de DNS automático, e por último reiniciar a nosa máquina

## comprobación

Unha vez realizado todos os pasos, lanzaremos o comando `docker compose up´, e se vamos ao navegador e buscamos o alias dunha das páxinas, no meu caso **www.fabulasmaravillosas.asircastelao.int** ou **www.fabulasoscuras.danielcastelao,int**, podremos observar o html que fixemos.

No caso de que ao lanzar o `docker compose up` da erro, pode ser porque estará apache2 iniciado, entón lanzaremos o comando `sudo systemctl stop apache2`.

