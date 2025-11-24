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

Una vez Enumerados todos los Servicios. Pasamos a intentar Ganar Acceso. Pareciera que el camino para lograrlo es intentar subir un archivo malicioso de alguna manera y, probando y probando, no lo conseguimos. Por lo tanto, pasamos a descargar los recursos PHP que son los que tienen la lógica de dicha función. Lo hacemos de la siguiente manera:  

* curl -O http://<IP>/file_upload.php
* curl -O http://<IP>/subir_archivo.php


___

#### FASE POST-EXPLOTACIÓN
