---
layout: post
title: Level1 - Crackmes
tags: [ctf, rev, crackmes]
---

**Autor**: sudo0x18 \\
**Dificultad**: 1/6

![img](/imgs/write-ups/crackmes/level1/level1.png)

## Intro

Este es un desafío bastante sencillo en el que hay que obtener la contraseña correcta en el binario.

## Análisis

Ejecuto el binario e introduzco un valor como prueba.

![img](/imgs/write-ups/crackmes/level1/level1_1.png)

Como se puede ver, es incorrecto.

## Radare2

Abro el binario con **radare2** y listo todas las funciones. Entre ellas hay que destacar **main** y **checkPass**.
![img](/imgs/write-ups/crackmes/level1/level1_2.png)

### Main

En la función **main** podemos ver que se llama a **checkPass** tras recibir una entrada.

![img](/imgs/write-ups/crackmes/level1/level1_3.png)

### CheckPass

En **checkPass** se puede ver como hay unas cuantas comprobaciones. Muy probablemente por sentencias "if" anidadas.
![img](/imgs/write-ups/crackmes/level1/level1_4.png)

Posible versión en python:
```py
#Comprobacion caracter por caracter
if password[0] == "s" and password[1] == "u" and password[2] == "d" and [...] :
    print("Correct")
else:
    print("Incorrect")
```

Utilizo **[Cyberchef](https://gchq.github.io/CyberChef/)** para convertir los valores decimales en caracteres y obtener la contraseña.
![img](/imgs/write-ups/crackmes/level1/level1_5.png)

## Comprobación

Introduzco la contraseña y efectivamente es correcta.
![img](/imgs/write-ups/crackmes/level1/level1_6.png)


Muchas gracias a **sudo0x18** por este desafío.
