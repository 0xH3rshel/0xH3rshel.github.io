---
layout: post
title: Switching - Crackmes
tags: [rev, crackmes, radare2, ghidra]
---

**Autor**: cat_puzzler \\
**Dificultad**: 2.5/6

![img](/imgs/write-ups/crackmes/switching/1.png)

# Objetivo

En esta entrada, resolveremos un crackme que nos reta a encontrar una contraseña válida. Utilizaremos herramientas como `Radare2` y `Ghidra` para analizar el binario y entender su lógica interna.

# Información

Abro el binario con `Radare2` y tras analizarlo muestro información del binario.


![img](/imgs/write-ups/crackmes/switching/2.png)

Se trata de un ejecutable `ELF de 64 bits` para sistemas `Linux`, compilado con `GCC 9.4.0` Presenta las siguientes características de seguridad:  
- **`Canary:`** Activado.  
- **`NX:`** Activado, lo que previene la ejecución en la pila.  
- **`RELRO:`** Completo, protegiendo las tablas de relocación.  

Además:  
- Es un binario dinámico (no estático), enlazado con **/lib64/ld-linux-x86-64.so.2**.  
- No está cifrado ni ofuscado (**crypto: false** y **stripped: false**).  
- Usa direcciones relativas (PIE activado).  
- Contiene símbolos y líneas de depuración (**lsyms: true**, **linenum: true**).  

Arquitectura: **AMD x86-64**, con formato **Little Endian**.  

Dado que en este caso no buscamos explotar el código, la característica más relevante es que `Stripped` está desactivado. Esto implica que el binario conserva información de depuración, como los nombres de las funciones, lo que facilita enormemente la tarea de analizar y comprender su funcionamiento.

# Funciones

Listo todas las funciones del programa y entre ellas hay varias que llaman la atención: `main`, `check_id_xor` y `check_id_sum`:

![img](/imgs/write-ups/crackmes/switching/3.png)

## Main

Comienzo analizando la función `main`. Dentro de esta función se están llevando a cabo varias comprobaciones, entre ellas mediante el uso de: 
 - **`strlen`**: Calcula la longitud de una cadena de caracteres hasta el carácter nulo (`\0`).  
 - **`strspn`**: Determina la longitud del segmento inicial de una cadena que contiene solo caracteres de un conjunto especificado.

En esta primera etapa, se verifica que al ejecutarse el binario, reciba un único parámetro, que será la clave a validar.

![img](/imgs/write-ups/crackmes/switching/4.png)

Después se realizan dos comprobaciones para verificar que la longitud de la clave introducida es mayor que 4 y menor que 255. En caso de no cumplirse, resultará en `error`.

![img](/imgs/write-ups/crackmes/switching/5.png)

Por último, se utiliza la función `strspn` para comprobar que todos los caracteres de la clave introducida pertenezcan al conjunto "0123456789", lo que garantiza que la clave esté compuesta exclusivamente por números. Seguidamente, se llaman las funciones `check_id_xor` y `check_id_sum`, y, según sus valores de retorno, se determina si la contraseña es válida o no.

![img](/imgs/write-ups/crackmes/switching/6.png)

A continuación se muestra la función **main** decompilada en `Ghidra`:

![img](/imgs/write-ups/crackmes/switching/7.png)

## Check Id Sum

Como se observa en la línea **36** de la imagen anterior, para que la contraseña sea válida, el valor de retorno de **check_id_sum** debe ser diferente de 0. Por ello, analizo esta función con el objetivo de determinar qué valor de entrada debe tener su parámetro para que su salida no sea 0.

![img](/imgs/write-ups/crackmes/switching/8.png)

Esta función realiza las siguiente comprobaciones: 
1. **Iteración sobre la clave**:  
   - Recorre la clave introducida completa mediante un índice. Comprueba si el índice actual (`passIndex`) ha alcanzado o superado la longitud de la cadena `PASS`. Si es así, finaliza el bucle (`while`).

2. **Diferenciación entre índices pares e impares**:  
   - Verifica si el índice actual (`passIndex`) es par o impar mediante la operación `(passIndex & 1)`.  
     - Si es par, toma el carácter correspondiente en `PASS` y lo convierte a número negativo.  
     - Si es impar, toma el carácter correspondiente en `PASS` y convierte a número positivo.

3. **Acumulación de valores**:  
   - Suma el valor calculado (positivo o negativo) al acumulador (`acumuladorSuma`).

4. **Comprobación del acumulador contra `xored_out`**:  
   - Verifica si el valor acumulado (`acumuladorSuma`) es igual a `xored_out` y si `xored_out` no es igual a `-1`.  
     - Si ambas condiciones son ciertas, la función retorna `1` (clave válida).  
     - En caso contrario, retorna `0` (clave no válida).

Ahora es necesario comprobar el funcionamiento de `check_id_xor` para asegurar que su valor de retorno es diferente de `-1`.

## Check Id Xor

En la siguiente imágen se muestra la función decompilada.

![img](/imgs/write-ups/crackmes/switching/9.png)

El funcionamiento es el siguiente:
1. **Inicialización de variables:**
   - `primerCharPass`: Se obtiene el primer carácter de la cadena `PASS`, convertido a entero.
   - `longitudPass`: Se calcula la longitud de la cadena `PASS` usando `strlen`.

2. **Cálculo inicial de `xored_out`:**
   - Se realiza una operación XOR entre:
     - `primerCharPass`.
     - El carácter final de la cadena `PASS` (`PASS[longitudPass - 1]` convertido a entero).
   - El resultado se guarda en `xored_out`.

3. **Comprobación de condiciones sobre `xored_out`:**
   - Se evalúan dos condiciones:
     - Si `xored_out` es menor que el valor de `primerCharPass`.
     - Si `xored_out` es menor que el último carácter de `PASS`.
   - Si cualquiera de estas condiciones se cumple, se asigna el valor `0xFFFFFFFF` a `xored_out`.

4. **Retorno del valor de `xored_out`:**
   - La función devuelve el valor final de `xored_out`.

# Conclusión

Este binario comprueba que:
 - La contraseña esté formada únicamente por números y tenga una longitud entre 4 y 255.
 - El valor de hacer la operación xor sobre le primer y el último caracter debe ser menor que ambos.
 - La suma de todos los caracteres (restando los pares y sumando los impares) debe ser igual al valor xor calculado previamente.

De esta manera, una posible clave es la siguiente:

```
$ ./crackme 011220
The password was correct
```

Comprobamos que `1-1+2-2 = 0^0 = 0`.

Y con esto ya estaría resuelto el crackme. Muchas gracias a `cat_puzzler` por este desafío.
