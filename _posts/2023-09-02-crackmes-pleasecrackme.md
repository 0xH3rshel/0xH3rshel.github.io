---
layout: post
title: PleaseCrackMe - Crackmes
tags: [ctf, rev, crackmes]
---

**Autor**: RaphDev \\
**Dificultad**: 1.4/6

![img](/imgs/write-ups/crackmes/pleasecrackme/pleasecrackme.png)

## Nota

El desafío es muy muy parecido a **trycrackme** que he resuelto anteriormente por lo que me voy a saltar bastantes pasos y centrarme en analizar como se calcula la contraseña.

## Comparación de la contraseña

Al igual que en **trycrackme** situo un breakpoint justo antes de que se ejecute la comprobación y al mostrar el contenido de los registros **rsi** y **rdi** podemos ver la contraseña. He introducido como nombre de usuario "user" y número "3" pero estos valores son arbitrarios. Lo importante es que la contraseña se generará a partir de ellos.

![img](/imgs/write-ups/crackmes/pleasecrackme/pleasecrackme_1.png)

## Calcular contraseña

Abro el binario en **Ghidra**, voy a la función main y renombro las variables para que sea más sencillo de entender.

![img](/imgs/write-ups/crackmes/pleasecrackme/pleasecrackme_2.png)

Como se puede ver, una vez introducidos el nombre de usuario y un número del 1 al 9, se realiza un bucle **while** que recorre todos los caracteres del nombre del usuario y le suma a cada uno el valor previamente introducido.

De esta forma se lleva a cabo una encriptación con el algoritmo [César](https://es.wikipedia.org/wiki/Cifrado_C%C3%A9sar).

Entendiendo el funcionamiento es muy sencillo encontrar una contraseña válida.

```
h3rshel@kali:~/Desktop/revs$ ./PleaseCrackMe
Type in your Username: a

Type in a number beetween 1 and 9: 1

Type in the password: b

You are succesfully logged in
```

Muchas gracias a **RaphDev** por este desafío.

