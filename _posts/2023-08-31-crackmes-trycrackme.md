---
layout: post
title: TryCrackMe - Crackmes
tags: [ctf, rev, crackmes]
---

**Autor**: MrEmpty \\
**Dificultad**: 1.7/6

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme.png)

## Intro

En este ejercicio el objetivo es conseguir una contraseña válida.

## Análisis

Abro el binario con **radare2**, obtengo información y listo las funciones.

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme_1.png)

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme_2.png)

### Main

En la siguiente imágen se puede ver un resumen de la función **main**.

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme_3.png)

Hay que destacar la comparación que se hace y tras la cual se muestra si la contraseña es válida o no.

Viendo parte de la función en mas detalle.

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme_4.png)

Como se puede apreciar, se ejecuta una llamada a la función **strcmp**, la cual comparará los valores de las cadenas a las que apuntan los registros **rdi** y **rsi**. En uno de estos registros se puede asumir que se guardará la entrada del usuario y en el otro la contraseña que necesitaremos obtener.

## Debug

Para poder leer la contraseña, situo un **breakpoint** justo antes de que se ejecute la comparación. Ejecuto el programa e introduzco una contraseña (0xH3rshel). Una vez el programa se detiene al alcanzar el breakpoint puedo imprimir las cadenas a las que apuntan los respectivos registros.

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme_5.png)

En este caso, mi entrada se ha guardado en el registro **rdi**, por lo que la contraseña debe ser la cadena que se ha almacenado en el **rsi**.

## Test

Ejecuto de nuevo el programa, pero ahora introduzco la cadena previamente obtenida.

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme_6.png)

Correcto :)

## Analisis en profundidad

Al principio de la función **main** se puede ver una cadena partida en 3 partes.

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme_7.png)

Al situar un **breakpoint** justo antes de que se ejecute la función **strlen()**, se puede ver la cadena completa en memoria.

Si nos fijamos un poco, el valor hexadecimal de esta cadena de texto, es la contraseña previamente obtenida.

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme_8.png)

Abro el programa en **cutter** donde se puede apreciar mejor el bucle **for**, el cual obtiene el valor hexadecimal de la cadena del principio del progama, la cual luego se comparará con la entrada del usuario.

![img](/imgs/write-ups/crackmes/trycrackme/trycrackme_9.png)

Y con esto ya estaría resuelto. :)

Muchas gracias a **MrEmpty** por este desafío.
