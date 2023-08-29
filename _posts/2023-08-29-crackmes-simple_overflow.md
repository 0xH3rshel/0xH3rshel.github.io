---
layout: post
title: Simple Overflow - Crackmes
tags: [ctf, rev, crackmes]
---

**Autor**: BitFriends \\
**Dificultad**: 1.3/6

![img](/imgs/write-ups/crackmes/simple-overflow/simple_overflow.png)

## Intro

En este ejercicio el objetivo es sobreescribir una variable mediante un bufferoverflow.

## Análisis

Lo primero que hago es abrir el binario con **radare2**, analizarlo y listar las funciones.

![img](/imgs/write-ups/crackmes/simple-overflow/simple_overflow_1.png)

## Main

La función **main** no hace nada a parte de llamar a **login**.

![img](/imgs/write-ups/crackmes/simple-overflow/simple_overflow_2.png)

## Login

He utilizado **Cutter** para decompilar esta función.

![img](/imgs/write-ups/crackmes/simple-overflow/simple_overflow_3.png)

Como se puede ver, utiliza la función **malloc()** para guardar en el heap dos variables.

- uVar1: Será donde fgets() guardará la entrada del usuario.
- uVar2: Es la variable que debemos sobreescribir.

Vemos que despues de guardar la entrada del usuario, comprueba si la variable **uVar2** es igual a 0. En caso de serlo, habremos conseguido iniciar sesión como admin.

## Explotación

Con **radare2** coloco un breakpoint justo después de la entrada el usurario.

![img](/imgs/write-ups/crackmes/simple-overflow/simple_overflow_4.png)

```
[0x5609cba57169]> db 0x5609cba571c0
```

A continuación, ejecuto el programa y le paso una entrada cualquiera.

![img](/imgs/write-ups/crackmes/simple-overflow/simple_overflow_5.png)

Como se puede ver en la imágen anterior, hay una distancia de 32 bytes entre el valor iniciar de **uVar2** (01) y la entrada del usuario.

### fgets()

La función que se utiliza para leer la entrada del usuario es **fgets()**, la cual leerá hasta que encuentre un salto de linea (\x0A). Por lo que será necesario mandarlo al final del exploit.

### uVar2

Esta variable es de tipo **int32_t** por lo que al menos necesitaremos 4 bytes contiguos de 0 (\x00).

## Exploit.py

```py
#!/bin/python3

from pwn import *

elf = ELF("./simple_overflow")
p = process(elf.path)

payload  =  b"0" * 32       # Padding
payload +=  b"\x00" * 4     # Override uVar2
payload +=  b"\x0a"         # EOL

p.send(payload)
print(p.recvline())
print(p.recvline())
```

Output:
```
┌──(h3rshel㉿kali)-[~/Desktop/revs]
└─$ python3 exploit.py
[*] '/home/h3rshel/Desktop/revs/simple_overflow'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
[+] Starting local process '/home/h3rshel/Desktop/revs/simple_overflow': pid 64946
[*] Process '/home/h3rshel/Desktop/revs/simple_overflow' stopped with exit code 0 (pid 64946)
b'enter password: uid: 0\n'
b'you are logged in as admin\n'
```

Muchas gracias a **BitFriends** por este desafío.