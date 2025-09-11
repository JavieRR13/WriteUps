# BORAZUWARAHCTF
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
   1   │ # Nmap 7.95 scan initiated Thu Sep 11 12:31:55 2025 as: /usr/lib/nmap/nmap --privileged -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG allPorts 172.17.0.2
   2   │ # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
   3   │ Host: 172.17.0.2 () Status: Up
   4   │ Host: 172.17.0.2 () Ports: 22/open/tcp//ssh///, 80/open/tcp//http///    Ignored State: closed (65533)
   5   │ # Nmap done at Thu Sep 11 12:31:59 2025 -- 1 IP address (1 host up) scanned in 4.03 seconds
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
   1   │ # Nmap 7.95 scan initiated Thu Sep 11 12:34:45 2025 as: /usr/lib/nmap/nmap --privileged -p22,80 -sCV -oN targeted 172.17.0.2
   2   │ Nmap scan report for 172.17.0.2
   3   │ Host is up (0.00s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
   7   │ | ssh-hostkey: 
   8   │ |   256 3d:fd:d7:c8:17:97:f5:12:b1:f5:11:7d:af:88:06:fe (ECDSA)
   9   │ |_  256 43:b3:ba:a9:32:c9:01:43:ee:62:d0:11:12:1d:5d:17 (ED25519)
  10   │ 80/tcp open  http    Apache httpd 2.4.59 ((Debian))
  11   │ |_http-server-header: Apache/2.4.59 (Debian)
  12   │ |_http-title: Site doesn't have a title (text/html).
  13   │ MAC Address: 02:42:AC:11:00:02 (Unknown)
  14   │ Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
  15   │ 
  16   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  17   │ # Nmap done at Thu Sep 11 12:35:06 2025 -- 1 IP address (1 host up) scanned in 21.46 seconds
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
Como podemos observar nos encontramos ante un servicio SSH con una versión actualizada por lo que lo dejaremos para más tarde. Mientras tanto me dirigiré al navegador web para examinar el servicio HTTP.

![Imagen Kinder](https://github.com/JavieRR13/WriteUps/blob/3995070a1ce6446556f883f9c8f6cc7851780fcb/DockerLabs/Muy%20f%C3%A1cil/%20BorazuwarahCTF/Im%C3%A1genes/BorazuwarahCTF_ImagenKinder.png)

Cuando abrimos la página nos encontraremos frente a una imagen de un huevo Kinder. Si miramos el código fuente lo único resaltable es dicha imagen.  Para continuar, probaremos con un poco de esteganografía así que descargaremos la imagen y la analizaremos con la herramienta [steghide](https://github.com/StegHigh/steghide).  Con esta herramienta podremos analizar datos ocultos dentro de la imagen.  
```ruby
❯ steghide extract -sf imagen.jpeg
Anotar salvoconducto: 
anot� los datos extra�dos e/"secreto.txt".
❯ ll
total 24K
-rw-rw-r-- 1 kali kali 19K sep 11 12:51 imagen.jpeg
-rw-rw-r-- 1 kali kali 104 sep 11 13:08 secreto.txt
```

* Con extract indicamos que queremos extraer el archivo oculto (si lo hay).
* Con -sf (stego file) indicamos el archivo que pensamos podría contener datos ocultos.

Vemos que nos ha descargado un archivo llamado *secreto.txt* que estaba oculto en la imagen.
```ruby
❯ cat secreto.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: secreto.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Sigue buscando, aquí no está to solución
   2   │ aunque te dejo una pista....
   3   │ sigue buscando en la imagen!!!
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Visto que no hemos obtenido mucha información de este fichero, otra opción es hacer uso de la herramienta [ExifTool](https://github.com/exiftool/exiftool) para analizar los metadatos que pueda contener.
```ruby
❯ exiftool imagen.jpeg
ExifTool Version Number         : 13.25
File Name                       : imagen.jpeg
Directory                       : .
File Size                       : 19 kB
File Modification Date/Time     : 2025:09:11 12:51:14+02:00
File Access Date/Time           : 2025:09:11 13:08:03+02:00
File Inode Change Date/Time     : 2025:09:11 12:51:14+02:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
XMP Toolkit                     : Image::ExifTool 12.76
Description                     : ---------- User: borazuwarah ----------
Title                           : ---------- Password:  ----------
Image Width                     : 455
Image Height                    : 455
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 455x455
Megapixels                      : 0.207
```

Podemos observar que el análisis nos ha reportado un nombre de usuario el cual podría ser válido para el servicio SSH que habíamos visto anteriormente.  Pero, para ello, primero debemos obtener el password.  Pa
