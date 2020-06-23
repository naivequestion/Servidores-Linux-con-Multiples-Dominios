# Servidores-Linux-con-Multiples-Dominios
Servidor DNS | Servidor Web | Servidor de Correo Electronico

## Tabla de contenido

- [Servidor DNS](#servidor-dns)
  - [Configuración de la IP estática](#configuración-de-la-ip-estática)
  - [Cambio de orden de resolución de búsqueda](#cambio-de-orden-de-resolución-de-búsqueda)
  - [Previo a la instalación del manejador de dns](#previo-a-la-instalación-del-manejador-de-dns)
  - [Instalación del manejador bind9](#instalación-del-manejador-bind9)
  - [Crear zonas](#crear-zonas)
  - [Verificar zonas (sintaxis)](#verificar-zonas-sintaxis)
  - [Consultas al DNS para verificar si está resolviendo](#consultas-al-dns-para-verificar-si-está-resolviendo)
  - [Configuración del host](#configuración-del-host)
  - [Otros comandos útiles](#otros-comandos-útiles)
- [Servidor Web](#servidor-web)
    - [Instalación de Apache](#instalación-de-apache)
    - [Crear la estructura de los directorios](#crear-la-estructura-de-los-directorios)
    - [Crear pagina web para cada dominio (host virtual)](#crear-pagina-web-para-cada-dominio-host-virtual)
    - [Crear nuevos archivos de host virtual](#crear-nuevos-archivos-de-host-virtual)
    - [Habilitar los nuevos archivos de host virtual](#habilitar-los-nuevos-archivos-de-host-virtual)
    - [Comandos útiles para pruebas apache](#comandos-útiles-para-pruebas-apache)
- [Servidor de Correo Electrónico](#servidor-de-correo-electrónico)
   - [Configuración del DNS para correo](#configuración-del-dns-para-correo)
    - [Creación de cuenta del sistema](#creación-de-cuenta-del-sistema)
    - [Instalación de postfix](#instalación-de-postfix)
    - [Configuración de postfix](#configuración-de-postfix)
        - [Cambio de formato para almacenar correos](#cambio-de-formato-para-almacenar-correos)
        - [Configuración de los dominios de buzones virtuales](#configuración-de-los-dominios-de-buzones-virtuales)
    - [Instalación de dovecot](#instalación-de-dovecot)
    - [Comandos útiles para pruebas](#comandos-útiles-para-pruebas)


## Servidor DNS

### Configuración de la IP estática
    
Lo siguiente es configurar el puerto de red y asignarle la IP que se ha definido para cada dominio en las zonas:

	nano /etc/network/interfaces
    
> Nota 1: se debe de utilizar ifconfig para copiar el nombre del adaptador al que se le van a asignar las IP estáticas. Si, las IP son estáticas por lo tanto si no se le configura una puerta de enlace como en este caso, entonces no habrá salida a internet.

> Nota 2: La dirección IP que se configure no debe de coincidir con la dirección predeterminada que se utiliza para configuración del router.

    # interfaces(5) file used by ifup(8) and ifdown(8)
    auto lo
    iface lo inet loopback

    auto enp2s0
    iface enp2s0 inet static
            address 192.168.1.100
            netmask 255.255.255.0
            dns-nameservers 192.168.1.100
            dns-search dom1.com

    auto enp2s0:0
    iface enp2s0:0 inet static
            address 192.168.1.101
            netmask 255.255.255.0
            dns-nameservers 192.168.1.101
            dns-search dom2.com

    auto enp2s0:1
    iface enp2s0:1 inet static
            address 192.168.1.102
            netmask 255.255.255.0
            dns-nameservers 192.168.1.102
            dns-search dom3.com

    auto enp2s0:2
    iface enp2s0:2 inet static
            address 192.168.1.103
            netmask 255.255.255.0
            dns-nameservers 192.168.1.103
            dns-search dom4.com


Se reinicia la interfaz que se ha modificado

	ip ad flush enp2s0
    ifdown enp2s0
    ifup enp2s0

> El archivo resolv.conf no va a mostrar la configuración apropiada hasta que no se reinicie máquina.

### Cambio de orden de resolución de búsqueda

Se debe de asegurar que primero se resuelva a través del dns y luego en el hosts.

	nano /etc/nsswitch.conf
    
La línea de hosts de quedar asi:

	hosts: dns files mdns4_minimal [NOTFOUND=return]

    
Finalmente se debe de reiniciar la maquinas y verificar 

	reboot
    cd /etc/resolv.conf

### Previo a la instalación del manejador de dns

En linux antes de cualquier instalación se debe de actualizar.

> Nota: En lo que resta del proceso se va a realizar todas las operaciones como super usuario.

	sudo su
    
	apt update
    
### Instalación del manejador bind9
    
	apt install bind9 bind9utils bind9-doc
    
### Crear zonas
    
Se van a poner las definiciones para las zonas (internas e inversas):

	nano named.conf.local
        
En este caso se van a definir 4 dominios, así que se tienen que definir 4 zonas. Dentro del archivo se agrega lo siguiente:

    // Definicion de la zona directa para el dominio dom1.com
    zone "dom1.com" {
        type master;
        file "/etc/bind/db.dom1.com";
    };

    // Definicion de la zona directa para el dominio dom2.com
    zone "dom2.com" {
        type master;
        file "/etc/bind/db.dom2.com";
    };

    // Definicion de la zona directa para el dominio dom3.com
    zone "dom3.com" {
        type master;
        file "/etc/bind/db.dom3.com";
    };

    // Definicion de la zona directa para el dominio dom4.com
    zone "dom4.com" {
        type master;
        file "/etc/bind/db.dom4.com";
    };

    // Definicion de la zona inversa para el dominio 192.168.1.1
    zone "1.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/db.1.168.192";
    };

    
Se procede a crear los archivos de las zonas que se han definido anteriormente (db.dom1.com,...,db.dom4.com).

> Nota 1: se recomienda editar utilizando nano que permite copiar y pegar.

> Nota 2: No se deben de utilizar espacios. Solo tabs.

	nano db.dom1.com
    nano db.dom2.com
    nano db.dom3.com
    nano db.dom4.com
    
Dentro del archivo **db.dom1.com** se agrega lo siguiente: 


	$TTL 86400
    @               IN      SOA     server.dom1.com. root.dom1.com. (
                                            0       ; Serial
                                            3600    ; Refresh
                                            1800    ; Retry
                                            604800  ; Expire
                                            86400   ; Negative Cache TTL
    )

                    IN      NS      server.dom1.com.
                    IN      A       192.168.1.100
    server          IN      A       192.168.1.100
    www             IN      CNAME   server

	
Dentro del archivo **db.dom2.com** se agrega lo siguiente:


    $TTL 86400
    @               IN      SOA     server.dom2.com. root.dom2.com. (
                                            0       ; Serial
                                            3600    ; Refresh
                                            1800    ; Retry
                                            604800  ; Expire
                                            86400   ; Negative Cache TTL
    )

                    IN      NS      server.dom2.com.
                    IN      A       192.168.1.101
    server          IN      A       192.168.1.101
    www             IN      CNAME   server



Dentro del archivo **db.dom3.com** se agrega lo siguiente:

    $TTL 86400
    @               IN      SOA     server.dom3.com. root.dom3.com. (
                                            0       ; Serial
                                            3600    ; Refresh
                                            1800    ; Retry
                                            604800  ; Expire
                                            86400   ; Negative Cache TTL
    )

                    IN      NS      server.dom3.com.
                    IN      A       192.168.1.102
    server          IN      A       192.168.1.102
    www             IN      CNAME   server



Finalmente dentro del archivo **db.dom4.com** se agrega lo siguiente:

    $TTL 86400
    @               IN      SOA     server.dom4.com. root.dom4.com. (
                                            0       ; Serial
                                            3600    ; Refresh
                                            1800    ; Retry
                                            604800  ; Expire
                                            86400   ; Negative Cache TTL
    )

                    IN      NS      server.dom4.com.
                    IN      A       192.168.1.103
    server          IN      A       192.168.1.103
    www             IN      CNAME   server


Se crea el archivo correspondiente a la zona inversa:

	nano db.1.168.192
    
Dentro de ese archivo se agrega lo siguiente:

    $TTL 86400
    @       IN      SOA     server.dom1.com.        root.dom1.com. (
                            0                       ; Serial
                            3600                    ; Refresh
                            1800                    ; Retry
                            604800                  ; Expire
                            86400                   ; Negative Cache TTL
    )
    			IN      NS      server.dom1.com.
    100       	IN      PTR     server.dom1.com.
    101       	IN      PTR     server.dom2.com.
    102       	IN      PTR     server.dom3.com.
    103       	IN      PTR     server.dom4.com.


### Verificar zonas (sintaxis)

Para verificar la sintaxis del archivo named.conf.local. Si no aparece ningún mensaje de errores en el promt significa que el archivo esta correcto.

	named-checkconf named.conf.local
    
Para verificar las zonas, si estan bien aparecerá un mensaje de OK.

	named-checkzone dom1.com db.dom1.com
    named-checkzone dom2.com db.dom2.com
    named-checkzone dom3.com db.dom3.com
    named-checkzone dom4.com db.dom4.com
    
    named-checkzone 1.168.192.in-addr.arpa db.1.168.192
    

Finalmente se debe de reiniciar el servidor

	service bind9 restart
    
Se verifica el status del servidor

	service bind9 status
    
### Consultas al DNS para verificar si está resolviendo

Se realiza un ping al servidor utilizando la IP

	ping -c 4 192.168.1.100
    ping -c 4 192.168.1.101
    ping -c 4 192.168.1.102
    ping -c 4 192.168.1.103
    
Se realiza un ping utilizando el host que en este caso es server

	ping -c 4 server

Se realiza utilizando el nombre completo (FQDN)

	ping -c 4 server.dom1.com
    ping -c 4 server.dom2.com
    ping -c 4 server.dom3.com
    ping -c 4 server.dom4.com
    
Pruebas con el *nslookup*

	nslookup server
    
    nslookup dom1.com
    nslookup dom2.com
    nslookup dom3.com
    nslookup dom4.com
    
    nslookup 192.168.1.100
    nslookup 192.168.1.101
    nslookup 192.168.1.102
    nslookup 192.168.1.103
    
Se limpia la cache

	rndc flush

    
### Configuración del host

Se abre el archivo hosts

	nano /etc/hosts
    
Dentro del mismo se escribe cada uno de los dominios con su correspondiente IP:

    127.0.0.1       localhost
    192.168.1.100   dom1.com server
    192.168.1.101   dom2.com server
    192.168.1.102   dom3.com server
    192.168.1.103   dom4.com server

    # The following lines are desirable for IPv6 capable hosts
    #::1     ip6-localhost ip6-loopback
    #fe00::0 ip6-localnet
    #ff00::0 ip6-mcastprefix
    #ff02::1 ip6-allnodes
    #ff02::2 ip6-allrouters
    
### Otros comandos útiles

	service networking restart
    service networking reload
    service networking status
    
## Servidor Web

### Instalación de Apache

	apt install apache2
    
### Crear la estructura de los directorios

Se crea una estructura de directorio que contendrá los datos del sitio que serviremos.

La raíz de nuestro documento (el directorio de nivel superior que Apache busca para encontrar contenido para servir) se configurará en directorios individuales en el directorio / var / www. Crearemos un directorio aquí para los cuatro hosts virtuales que se realizarán.

Dentro de cada uno de estos directorios, se crea una carpeta public_html que contendrá los archivos del sitio web. 

	mkdir -p /var/www/dom1.com/public_html
    mkdir -p /var/www/dom2.com/public_html
    mkdir -p /var/www/dom3.com/public_html
    mkdir -p /var/www/dom4.com/public_html
    
También se debe modificar los permisos para garantizar que se permita el acceso de lectura al directorio web general y a todos los archivos y carpetas que contiene para que las páginas se puedan servir correctamente:

	chmod -R 755 /var/www
    
### Crear pagina web para cada dominio (host virtual)

Se crea un documento .html dentro de cada uno de los directorios que se crearon anteriormente:

    nano /var/www/dom1.com/public_html/index.html
    nano /var/www/dom2.com/public_html/index.html
    nano /var/www/dom3.com/public_html/index.html
    nano /var/www/dom4.com/public_html/index.html
    
Posteriormente dentro de cada archivo html se escribe el contenido que se desea mostrar en los sitios.

	<!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Bienvenido</title>
      </head>
      <body>
        <h1>Dominio 1</h1>
        <h2>Servidor web cargado con exito!</h2>
        <p>redes.dom1.com - 192.168.1.1</p>
      </body>
    </html>
    
### Crear nuevos archivos de host virtual


Los archivos de host virtual son los archivos que especifican la configuración real de nuestros hosts virtuales y determinan cómo responderá el servidor web Apache a varias solicitudes de dominio.

Apache viene con un archivo de host virtual predeterminado llamado 000-default.conf que podemos usar como punto de partida. Lo copiaremos para crear un archivo de host virtual para cada uno de nuestros dominios.

	cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/dom1.com.conf
    
Se abre el archivo con algún editor:

	nano /etc/apache2/sites-available/dom1.com.conf
    
Primero, se debe cambiar la directiva ServerAdmin a un correo electrónico a través del cual el administrador del sitio pueda recibir correos electrónicos.

	ServerAdmin root@dom1.com
    
Después, se necesita agregar dos directivas. La primera, llamada ServerName, establece el dominio base que debe coincidir con esta definición de host virtual. Este será su dominio. El segundo, llamado ServerAlias, define otros nombres que deberían coincidir como si fueran el nombre base.

	ServerName dom1.com
	ServerAlias www.dom1.com
    

Luego se debe de modificar es la ubicación de la raíz del documento para este dominio:

	DocumentRoot /var/www/dom1.com/public_html
    
Removiendo los comentarios se veria asi:

	 <VirtualHost 192.168.1.100:*>
        ServerName dom1.com
        ServerAdmin root@dom1.com
        ServerAlias www.dom1.com
        DocumentRoot /var/www/dom1.com/public_html
        DirectoryIndex index.html
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
	</VirtualHost>
    
> Nota: Este proceso se debe de realizar para cada uno de los dominios.

### Habilitar los nuevos archivos de host virtual

Ahora que se han creado los archivos de host virtuales, debemos habilitarlos, para ello apache incluye algunas herramientas que nos permiten hacer esto.Utilizaremos la herramienta a2ensite para habilitar cada uno de los nuestros sitios.

	a2ensite dom1.com.conf
    a2ensite dom2.com.conf
    a2ensite dom3.com.conf
    a2ensite dom4.com.conf
    
Lo siguiente es deshabilitar el sitio predeterminado definido en 000-default.conf:

	a2dissite 000-default.conf
    
Se debe reiniciar Apache para que estos cambios surtan efecto y usar el estado systemctl para verificar el éxito del reinicio.

	systemctl restart apache2
	systemctl status apache2

Lo último es dirigirse al navegador y acceder utilizando los nombres de dominio y las direcciones IP.

### Comandos útiles para pruebas apache

Logs de apache:

	tail -f /var/log/apache2/error.log

## Servidor de Correo Electrónico

Cuando se está trabajando con un servidor de correo electrónico con múltiples dominios, una opción clara es utilizar buzones virtuales.

### Configuración del DNS para correo

Hay que editar cada una de las zonas que que fueron creadas durante la configuración del DNS.

Dentro de estas le debe de indicar un registro para resolver DNS, al que en este caso le llamaremos **correo**. Luego se debe de agregar un registro de correo, con el cual indicamos que los correos que entren con la dirección *@dom1.com, @dom2.com, @dom3.com, @dom4.com* sean redirigidos a **correo**:

db.dom1.com 

    correo          IN      A       192.168.1.100
	dom1.com.       IN      MX 10   correo
    


db.dom2.com

	correo          IN      A       192.168.1.101
    dom2.com.       IN      MX 10   correo
    
db.dom3.com

	correo          IN      A       192.168.1.102
    dom3.com.       IN      MX 10   correo

db.dom4.com

	correo          IN      A       192.168.1.103
    dom4.com.       IN      MX 10   correo   

Después de guardar los cambios y comprobar que no hayan errores de sintaxis en las zonas, para ello:

    named-checkzone dom1.com /etc/bind/db.dom1.com
    named-checkzone dom2.com /etc/bind/db.dom2.com
    named-checkzone dom3.com /etc/bind/db.dom3.com
    named-checkzone dom4.com /etc/bind/db.dom4.com

Lo siguiente a hacer es configurar la zona inversa, para que otros servidores de correo no clasifiquen los correos enviados como spam.

	nano /etc/bind/db.1.168.192

Al final del archivo se agrega:

    100     IN      PTR     correo.dom1.com.
	101     IN      PTR     correo.dom2.com.
    102     IN      PTR     correo.dom3.com.
    103     IN      PTR     correo.dom4.com.
    
Finalmente se reinicia el bind9 y se comprueba su estado.

	service bind9 restart
   	service bind9 status
    
Con nslookup, se comprueba que el servidor esté resolviendo a correo.

    nslookup correo.dom1.com
    nslookup correo.dom2.com
    nslookup correo.dom3.com
    nslookup correo.dom4.com

También se hace un nslookup inverso

	nslookup 192.168.1.100
    nslookup 192.168.1.101
    nslookup 192.168.1.102
    nslookup 192.168.1.103


### Creación de cuenta del sistema

Se necesita configurar una cuenta del sistema, para que esta actúe como propietario de todos los buzones virtuales. Eso implica que los buzones de cada dominio se organizarán bajo el directorio de inicio de esta cuenta del sistema:

> La representación siguiente muestra como quedaría la estructura para una cuenta de sistema llamada "vmail".

    
	    /home/vmail/
	    |-- dom1.com/
	    |	  `--  info/
	    |           |-- new/
	    |	        |-- cur/
	    |           `-- tmp/
	    |-- dom2.com/
	    |	  `--  info/
	    |           |-- new/
	    |	        |-- cur/
	    |           `-- tmp/
	    .
	    .
	    .

Se procede a crear la cuenta y grupo:

	groupadd -g 5000 vmail
    useradd -u 5000 -g vmail -s /usr/bin/nologin -d /home/vmail -m vmail
    
> Se usa un gid y uid de 5000 en ambos casos para que no tengamos conflictos con los usuarios habituales.

> Se puede cambiar el directorio de inicio, por ejemplo a */var/mail/vmail*

### Instalación de postfix

Postfix es un agente de transferencia (MTA), el cual se encarga de transferir correos utilizando el protocolo SMTP. Además se postfix se puede utilizar el conjunto de utilidades *Mailutils* que permite la gestión del correo electronico desde la linea de comandos:

	apt install postfix postfixadmin dbconfig-no-thanks mailutils
    
1. En el tipo de configuración se debe de escoger **Sitio de Internet**.
2. En el nombre de sistema de correo se debe de poner el nombre del servidor:

> Cuando se tienen múltiples dominios como en este caso se debe de poner al que le fue asignado el registro NS en la zona inversa del DNS.  

### Configuración de postfix

Antes de modificar los archivos de postfix, **es recomendable** hacer una copia de estos:

	cp /etc/postfix/main.cf /etc/postfix/main.cf.orig
    cp /etc/postfix/master.cf /etc/postfix/master.cf.orig
    
Posteriormente se procede a configurar el archivo main.cf:

	nano /etc/postfix/main.cf
    
Lo primero que hay que editar, es el parámetro *myhostname*, y se le debe de asignar como prefijo el nombre que se le asignó al registro MX del DNS:

	myhostname = correo.dom1.com
    
Lo siguiente es agregar las direcciones IP desde las que será permitido enviar correos, en este caso se va a agregar antes de la dirección de localhost:

	mynetworks = 192.161.1.0/24 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
    
#### Cambio de formato para almacenar correos

Postfix y Mailutils de forma predeterminada utilizan el *formato extendido para almacenar correos* **mbox** el cual guarda los correos en un solo archivo. Existe otro formato que se llama **Maildir** el cual utiliza un directorio, con subdirectorios, para guardar los mensajes en ficheros individuales. Para esta configuración es este último el que se va a utilizar. Para ello se agrega el parámetro *home_mailbox*:

	home_mailbox = Maildir/

#### Configuración de los **dominios de buzones virtuales**

Dentro del mismo archivo (main.cf) al final se debe de agregar la ruta al fichero que contendrá los dominios desde los que se podrá enviar correos:

	virtual_mailbox_domains = /etc/postfix/vhosts
    
> Los dominios también se pueden agregar directamente. Por ejemplo: virtual_mailbox_domains = dom1.com dom2.com...

El siguiente parámetro a agregar, *virtual_mailbox_base* es un prefijo que el agente de entrega virtual (postfix) antepone para todos los nombres de los buzones virtuales.

> Por ejemplo: /home/vmail/dom2.com

Entonces, en este caso '/home/vmail' es el directorio base donde se almacenará nuestro correo.

	virtual_mailbox_base = /home/vmail

Se agrega el parámetro *virtual_mailbox_maps* en el cual especificamos el archivo que actuará como tabla de búsqueda para las direcciones de correo electrónico. Ese archivo es un simple archivo de texto de dos columnas. En la primera columna, se especifica la dirección de correo electrónico y en la segunda se define la ubicación del buzón correspondiente.

	virtual_mailbox_maps = hash:/etc/postfix/vmaps
    
Los siguientes dos parámetros *virtual_uid_maps* y *virtual_gid_maps* determinan el propietario y el grupo que Postfix usa cuando realiza entregas a archivos de buzones virtuales.

	virtual_uid_maps = static:5000
    virtual_gid_maps = static:5000
    
> En este caso se le asigna el identificador de usuario y de grupo que le fue asignado a la cuenta del sistema que se creó anteriormente.

El siguiente parámetro que se debe de agregar es *virtual_minimum_uid* con el cual se especifica el identificador de usuario mínimo que postfix aceptara.

	virtual_minimum_uid = 1000
    
Se debe de especificar el transporte virtual predeterminado para para entregas de mensajes a direcciones de buzones virtuales que en este caso será dovecot:

	virtual_transport = dovecot
    
>  Si se desea probar el funcionamiento de postfix antes de instalar y configurar dovecot, se debe de comentar este parámetro.

Para indicar el limite de almacenamiento que tendrán los usuarios.	
    
    mailbox_size_limit = 0
    
> 0 significa que por ejemplo info@dom1.com tiene almacenamiento ilimitado para su buzon.

Antes de finalizar la configuracion dentro de main.cf se debe de agregar el parametro *recipient_delimiter*

    recipient_delimiter = +

Lo siguiente es crear el archivo que le fue asignado al parámetro *virtual_mailbox_domains*:

	nano /etc/postfix/vhosts
    
Como se mencionó, dentro de este se deben de poner los dominios (existentes):

	dom1.com
    dom2.com
    dom3.com
    dom4.com
    
> En caso se utilizar una base de datos, es en este archivo en el que se debe de hacer una consulta a alguna tabla dentro de la base de datos que contenga los dominios.

Luego se debe de crear el archivo que le fue asignado a *virtual_mailbox_maps*:
    
    nano /etc/postfix/vmaps
    
Dentro de este se debe de poner lo siguiente:

	info@dom1.com   dom1.com/info/
    dev@dom1.com    dom1.com/dev/
    team@dom1.com   dom1.com/team/
    info@dom2.com   dom2.com/info/
    dev@dom2.com    dom2.com/dev/
    team@dom2.com   dom2.com/team/
    info@dom3.com   dom3.com/info/
    dev@dom3.com    dom3.com/dev/
    team@dom3.com   dom3.com/team/
    info@dom4.com   dom4.com/info/
    dev@dom4.com    dom4.com/dev/
    team@dom4.com   dom4.com/team/

> Es importante el / al final de la ruta de los directorios para que postfix entienda de que se está utilizando el formato Maildir.

> De igual forma, se estuviera utilizando una base de datos, es dentro de este archivo que se escribiría la consulta para alguna tabla que contenga los usuarios (correos).

Lo siguiente es indicar con el comando postmap que se cree la tabla de búsqueda de los correos:

	postmap /etc/postfix/vmaps
    
> Nota: Cada vez que se modifica este archivo se debe de aplicar el comando postmap.

Dentro del archivo **/etc/postfix/master.cf** se debe de agregar lo siguiente al final del mismo:

	dovecot   unix  -       n       n       -       -       pipe
      flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -f ${sender} -d ${user}@${nexthop}

> La la indentación antes de "flags" es sumamente importante.

> Si se desea probar postfix antes de haber instalado dovecot se debe de comentar estas dos líneas.

Para verificar que no hayan errores en los archivos modificados anteriormente, se utiliza el comando:

	postfix check

Si el comando no retorna nada significa que todo está bien, sin embargo, si este retorna el siguiente mensaje:

	postfix/postfix-script: fatal: cannot execute /usr/sbin/postconf!
    
Significa que hay un error de sintaxis, probablemente de indentación.

Finalmente se debe de reiniciar postfix y verificar el estado del mismo:

	service postfix restart
    service postfix status
    
### Instalación y configuración de dovecot

Dovecot es un agente de reparto (MDA), y este lo que hace es pasar el correo desde el servidor a usuarios (clientes como Thunderbird) utilizando el protocolo IMAP o el protocolo POP3.

	apt update
    apt install dovecot-core dovecot-imapd

Antes de iniciar a modificar los archivos de configuración de dovecot es **necesario** hacer copias de seguridad.
    
    cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.orig
    cp /etc/dovecot/conf.d/10-mail.conf /etc/dovecot/conf.d/10-mail.conf.orig
    cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.orig
    cp /etc/dovecot/conf.d/auth-static.conf.ext /etc/dovecot/conf.d/auth-static.conf.ext.orig 
    
En esta primera etapa no se utiliza autenticación SSL/TLS, para ello se debe de permitir que las cuentas se puedan autenticar utilizando texto plano, para ello se debe de abrir el archivo 10-auth.conf:

	nano /etc/dovecot/conf.d/10-auth.conf

Hay que descomentar *disable_plaintext_auth* y asignarle *no* y asegurarse de que *auth_mechanisms* este descomentado:

	disable_plaintext_auth = no
    auth_mechanisms = plain

Finalmente se debe de descomentar el siguiente include:

    !include auth-static.conf.ext

El siguiente archivo que se editará es *10-mail.conf*:
	
    nano /etc/dovecot/conf.d/10-mail.conf
    
Se deben de documentar y configurar los siguientes valores:

	mail_location = maildir:/home/vmail/%d/%n
    mail_uid = 5000
	mail_gid = 5000
    mail_privileged_group = vmail
    
> mail_location está haciendo referencia a la ruta de un usuario n, con dominio d, dentro de /home/vmail/.
    
Se debe de comentar:

	mail_location = mbox:~/mail:INBOX=/var/mail/%u

En el archivo *10-ssl.conf* se debe de modificar el soporte a ssl:

	ssl = no

Debido a que se están utilizando cuentas virtuales y no cuentas del sistema, se deben de incluir las cuentas de correos y sus respectivas credenciales en algún archivo que contenga la misma estructura que la del sistema. Para ello, se debe de modificar el archivo *auth-static.conf.ext*:

	nano /etc/dovecot/conf.d/auth-static.conf.ext
    
En este archivo se van a hacer los llamados al archivo que va a contener las contraseñas de los correos y los usuarios.

	passdb {
      driver = passwd-file
      args = scheme=plain-md5 /etc/dovecot/passwd
    }

    userdb {
      driver = static
      args = /etc/dovecot/users
    }
    
Se procede a crear el archivo *users*:

	nano /etc/dovecot/users

> Este contiene la misma estructura que el archivo */etc/shadow*.

	info@dom1.com::5000:5000::/home/vmail/dom1.com/:/bin/false::
    dev@dom1.com::5000:5000::/home/vmail/dom1.com/:/bin/false::
    team@dom1.com::5000:5000::/home/vmail/dom1.com/:/bin/false::
    info@dom2.com::5000:5000::/home/vmail/dom2.com/:/bin/false::
    dev@dom2.com::5000:5000::/home/vmail/dom2.com/:/bin/false::
    team@dom2.com::5000:5000::/home/vmail/dom2.com/:/bin/false::
    info@dom3.com::5000:5000::/home/vmail/dom3.com/:/bin/false::
    dev@dom3.com::5000:5000::/home/vmail/dom3.com/:/bin/false::
    team@dom3.com::5000:5000::/home/vmail/dom3.com/:/bin/false::
    info@dom4.com::5000:5000::/home/vmail/dom4.com/:/bin/false::
    dev@dom4.com::5000:5000::/home/vmail/dom4.com/:/bin/false::
    team@dom4.com::5000:5000::/home/vmail/dom4.com/:/bin/false::


Luego se crea el archivo *passwd*:

	nano /etc/dovecot/passwd
    
Este archivo contiene las contraseñas encriptadas con el esquema que fue definido en el archivo *auth-static.conf.ext*, en este caso, MD5:

	info@dom1.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    dev@dom1.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    team@dom1.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    info@dom2.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    dev@dom2.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    team@dom2.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    info@dom3.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    dev@dom3.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    team@dom3.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    info@dom4.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    dev@dom4.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.
    team@dom4.com:{MD5}$1$FAJbEQsW$DPA8yC3yLzizYoyuxMK.9.


Para generar contraseñas encriptadas:

	doveadm pw -s md5
o

    doveadm pw -s sha256
    doveadm pw -s sha512
    
> En este caso se está utilizando las misma contraseña para las cuentas.

Estos archivos actualmente son propiedad de root, sin embargo, se tiene que asegurar que que el usuario dovecot que se crea de forma automática tenga acceso a estos archivos:

	chown uid_dovecot:gid_dovecot /etc/dovecot/users
    chown uid_dovecot:gid_dovecot /etc/dovecot/passwd
    
Para obtener el uid y gid de dovecot:

	cat /etc/passwd | grep dovecot

Finalmente, se tiene que asegurar de habilitar SMTP e IMAP a través del firewall:

	apt install firewalld
    
    firewall-cmd --add-port=143/tcp
    firewall-cmd --add-port=143/tcp --permanent
    firewall-cmd --add-port=110/tcp
    firewall-cmd --add-port=110/tcp --permanent
    firewall-cmd --add-port=587/tcp
    firewall-cmd --add-port=587/tcp --permanent

Se debe de recargar y verificar el estado de dovecot:

	service dovecot reload
    service dovecot status
    
### Comandos útiles para pruebas

Logs de apache:

	tail -f /var/log/apache2/error.log
    
Logs de postfix:

	tail -f /var/log/mail.log

Logs de dovecot:

    tail -f /var/log/dovecot.log 

### Referencias

- [BIND9ServerHowto](https://help.ubuntu.com/community/BIND9ServerHowto)
- [DNS and BIND](https://books.google.com.ni/books?id=Xb9O5yNS5GEC&printsec=frontcover#v=onepage&q&f=false)
- [Apache IP-based Virtual Host Support](https://httpd.apache.org/docs/2.4/vhosts/ip-based.html)
- [VirtualHost Examples](https://httpd.apache.org/docs/2.4/vhosts/examples.html)
- [Postfix: The Definitive Guide](https://books.google.com.ni/books?id=tdObAgAAQBAJ&printsec=frontcover&dq=inauthor:%22Kyle+D.+Dent%22&hl=es-419&sa=X&ved=2ahUKEwiy4eu40ZjqAhUJneAKHW3pCOMQ6AEwAHoECAEQAg#v=onepage&q&f=false)
- [Virtual user mail system with Postfix, Dovecot and Roundcube](https://wiki.archlinux.org/index.php/Virtual_user_mail_system_with_Postfix,_Dovecot_and_Roundcube)
- [Postfix Standard Configuration Examples](http://www.postfix.org/STANDARD_CONFIGURATION_README.html)
- [Postfix Configuration Parameters](http://www.postfix.org/postconf.5.html)
- [Postfix Virtual Domain Hosting Howto](http://www.postfix.org/VIRTUAL_README.html)
- [Dovecot LDA with Postfix](https://wiki.dovecot.org/LDA/Postfix)
- [Passwd-file](https://doc.dovecot.org/configuration_manual/authentication/passwd_file/)
- [Password Schemes](https://doc.dovecot.org/configuration_manual/authentication/password_schemes/#authentication-password-schemes)
- [User Databases (userdb)](https://doc.dovecot.org/configuration_manual/authentication/user_databases_userdb/#authentication-user-database)

