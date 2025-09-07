# INJECTION
### INDICACIONES PREVIAS
* Sistema operativo: Kali Linux
* Terminal: ZSH Powerlevel10K
* Utilidad mkt definida en la zshrc para la creación automática de directorios nmap, content y scripts.
* Cat aliseado como bat.
## DESPLIEGUE DE MÁQUINA
1. Descargarla desde la plataforma DockerLabs.
2. En mi caso, yo me creo un directorio con el nombre de la máquina para tenerlo todo más organizado, por lo que mediante el comando mv (`mv ~/Downloads/<máquina.zip> ~/Desktop/Máquinas/Dockerlabs/<máquina>`) la moveré de descargas a la nueva ubicación.
3. Una vez hecho esto, la descomprimiremos mediante unzip <máquina.zip> y se nos creará un ejecutable .tar
4. Por último, la desplegaremos mediante sudo bash auto_deploy.sh <máquina.tar> y se nos proporcionará una IP, en mi caso 172.17.0.2.
### WRITEUP
Primero comprobaremos la conectividad con la máquina. 
```
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

``❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts``
* Con -p- escaneamos el rango completo de puertos (65535).
* Con --open añadimos que de esos solo nos interesan los abiertos.
* Con -sS seleccionamos que el tipo de escaneo será de tipo SYN Scan. Este se caracteriza por ser rápido y sigiloso puesto que no se completa el three-way handshake.
  * Según la respuesta del puerto, Nmap deduce su estado:
    * SYN/ACK recibido → el puerto está abierto.
    * RST (Reset) recibido → el puerto está cerrado.
    * Sin respuesta o ICMP error → el puerto está filtrado (por firewall).
  * Si el puerto estaba abierto (SYN/ACK), Nmap no completa la conexión: envía un RST para “cortar” el handshake y no dejar un rastro evidente.
* Con --min-rate 5000 indicamos que queremos un envío de 5000 paquetes por segundo.
* Con -vvv (triple verbose) se muestra por consola la información detallada y actualizada al momento.
* Con -n indicaremos que no queremos resolución DNS, trabajaremos directamente con direcciones IP.
* Con -Pn queremos que no se haga ping al host antes de comenzar el escaneo, asumiendo que está activo.
* Por último, con -oG exportamos la evidencia en un archivo con formato grepeable para facilitar su procesamiento en auditorias.

Para simplificarnos el visionado de los puertos abiertos directamente abriremos el archivo "allPorts".
```
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

``❯ nmap -p22,80 -sCV 172.17.0.2 -oN targeted``

*El parámetro -sCV es una conjunción de los parámetros:
  * -sC: Ejecuta los scripts de Nmap por defecto (Default Scripts) para el host/puerto para detectar vulnerabilidades básicas, información de banners, servicios públicos, etc..
  * -sV: Detecta la versión del servicio que corre en cada puerto abierto.
* Con -oN exportamos la evidencia en un archivo con formato Nmap para dejarlo registrado.

Al igual que antes, para que la visualización de toda la información sea más cómoda, la analizaremos desde el archivo que acabamos de generar.
```
❯ cat targeted
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

