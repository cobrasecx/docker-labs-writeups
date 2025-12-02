# WHOIAM

#### Dificultad: Fácil

___

#### FASE RECONOCIMIENTO:

1. Escaneo:  
Usaremos `nmap -n -v --open -sV -sV -oN escaneo <IP>`.  
Del mismo obtenemos que únicamente el puerto 80 tiene un servicio. por lo tanto se trata de una Web-App y que la versión es Apache 2.4.58 y que el sistema host en un sistema Ubuntu.

Ahora, prodríamos correr también `whatweb`: `whatweb <IP>` para Footprinting.  


2. Enumeración Web:  

2.1. Empezamos con los Directorios, usando, por ejemplo, `wfuzz -c --hc 404 -t 200 -z file,<diccionario> -R 3 -u "http://<IP>/FUZZ"` encontramos:  
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

2.2 Ahora proseguimos con los Recursos (PHP-HTML-TXT):  
`wfuzz -c --hc 404 -t 200 -z file,/usr/share/wordlists/dirb/big.txt -z list,"php-html-txt`  

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

De estas enumeración, por el momento nos quedamos con `/backups`. Yendo a este endpoint vemos que podemos descargar un recurso: `databaseback2may.zip`. Lo hacemos y primero revisamos localmente sin extraer para estar seguros. Lo hacemos con `unzip -l <zip>`. Vemoos muestra un único archivito llamado `29DBMay`. Ahora sí, procedemos a extraerlo con `unzip <zip>` le hacemos un `cat 29DBMay` y obtenemos las credenciales `developer:2wmy3KrGDRD%RsA7Ty5n71L^`. Investigando un poco con `hashid 2wmy3KrGDRD%RsA7Ty5n71L^` y la web vemos que no se trata de una hash válido ni base64, entonces probablemente se trata de un texto claro.  

Entonces, procedemos a probar nuestras nuevas credenciales en el endpoint `/wp-login.php` y tenemos Footholding Web.
4.  


#### FASE EXPLOTACIÓN:
#### FASE POST-EXPLOTACIÓN: Elevar Privilegios

#### MITIGACIONES:  
