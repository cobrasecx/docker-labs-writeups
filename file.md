# File  

> Dificultad: Fácil

___

#### FASE ENUMERACIÓN  

Primero hacemos un Escaneo de Objetivo. Para ello, usamos nmap:  
`nmap -v -n --open -sV -sC --min-rate 5000 <IP> -oN escaneo`  

Vemos que el puerto 80 y 21 están abiertos. Vamos a Enumerar ambos servicios.  

Vemos que el servicio FTP admite usuario Anónimo, por lo cual corremos: `ftp <IP>` e ingresamos `Anonymous` en el campo de usuario. Al pedir la password, sólo presionamos Enter y listo, estamos dentro del servidor FTP. Ahora lanzamos `dir` o `ls` y vemos el contenido del directorio `/var/ftp/`.  

Vemos que hay un archivo interesante llamado "anon.txt" el cual descargamos localmente con `get anon.txt`. Ahora, en nuestro equipo, hacemos un `cat anon.txt` y nos devuelve un hash `53dd9c6005f3cdfc5a69c5c07388016d`. Para saber primero el tipo de hash corremos `hashid <hash>` en nuestro Linux y notamos que es un `SHA1` el cual es muy poco seguro y fácil de crackear.  

Entonces ahora, vamos a la página https://crackstation.net/, ingresamos nuestro hash y obtenemos que descodificado es `justin`. Por el momento, eso es todo con respecto a FTP.  

Ahora Enumeremos el servicio Web:  





___

#### FASE EXPLOTACIÓN  
___

#### FASE POST-EXPLOTACIÓN
