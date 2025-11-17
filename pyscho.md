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

