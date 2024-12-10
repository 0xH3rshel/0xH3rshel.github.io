---
layout: post
title: Keygen Lab - Writeup
tags: [rev, radare, asm]
---

Autor: 0xH3rshel

# Introducción

En este post voy a resolver el laboratorio **Introductory Keygen** del libro X86 Software Reverse Engineering: Cracking and Counter-Measures.

![img](/imgs/write-ups/reversing/rebingo/portada.jpg#center)

El objetivo principal es averiguar el algoritmo empleado para determinar la validez de las claves proporcionadas. Una vez comprendido el funcionamiento interno del binario y revelado desarrollo un generador de claves capaz de producir entradas válidas. 


# Lab Introductory Keygen

## Análisis estático

Abro el binario en **radare2** y comienzo el análisis estático.

![img](/imgs/write-ups/reversing/keygen/1.png#center)

Identifico que se trata de un ejecutable en formato **ELF32** diseñado para la arquitectura **x86** (Intel 80386) en modo **Little Endian** y con una longitud de palabra de **32 bits**. Fue compilado con **GCC 4.8.4** para sistemas operativos basados en **Linux**, utilizando como intérprete dinámico `/lib/ld-linux.so.2`.

En cuanto a las medidas de seguridad, el binario cuenta con protección **Canary** para prevenir desbordamientos de pila y soporte **NX** para evitar la ejecución de código en áreas de memoria marcadas como no ejecutables. Sin embargo, no está compilado con soporte **PIE**, lo que significa que las direcciones no son aleatorias, facilitando la tarea de ingeniería inversa. Además, dispone de una protección **Relro parcial**, ofreciendo una protección limitada contra ataques de sobrescritura en la tabla GOT.

Por último, es importante destacar que el binario no está **stripped**, por lo que conserva los símbolos y números de línea. Esto es particularmente útil, ya que permite realizar un análisis más detallado del código sin necesidad de reconstruir información perdida.


![img](/imgs/write-ups/reversing/keygen/2.png#center)

El análisis de cadenas revela que el programa solicita un **nombre** y una **clave** en formato específico (`xxxxx-xxxxx-xxxxx-xxxxx`). Según la entrada, responde con mensajes que confirman si la clave es válida o no. Esto sugiere que el binario implementa un sistema de validación que debemos analizar para entender su funcionamiento y lógica.

![img](/imgs/write-ups/reversing/keygen/3.png#center)

El binario tiene dos funciones clave: `main`, que gestiona el flujo principal, y `sym.key_check`, que realiza la validación de la clave. Esta última será el foco principal del análisis para entender el algoritmo de verificación.

### Main

Esta función no presenta ninguna otra función más que llamar a sym.key_check, en la que se realizará toda la lógica del programa.

![img](/imgs/write-ups/reversing/keygen/4.png#center)

### Key_check

El gráfico de flujo muestra la lógica de la función `key_check`, donde se evalúan múltiples condiciones en serie. Cada nodo representa una comparación o decisión, y el flujo se divide en ramas dependiendo de si las condiciones se cumplen o no. Finalmente, se llega a un nodo que determina si la clave es válida o no.

![img](/imgs/write-ups/reversing/keygen/5.png#center)

#### Comprobaciones

En esta sección del código se identifican dos tipos de comprobaciones:

![img](/imgs/write-ups/reversing/keygen/7.png#center)

1. **Comprobación de longitud** (bloque rojo): Se utiliza la función `strlen` para verificar que la longitud de la cadena `nombre` es al menos 8 caracteres. Si no se cumple, el flujo del programa salta directamente a un punto de fallo.

2. **Instrucciones y validación adicional** (bloque azul): Esta comprobación se realiza 4 veces durante la función. Si alguna de estas validaciones falla, el flujo también salta a un punto de error. Se puede apreciar que en estos bloques se realizan un conjunto de instrucciones simples: movimientos, multiplicaciones y comparaciones seguidas de su correspondiente salto condicional. Es fácil deducir que se van multiplicar dos valores y si son iguales a otro, la ejecución continua. De lo contrario, salta al final del programa y la ejecución termina.

El programa asegura primero una longitud mínima para la entrada y luego verifica que cumpla con ciertos criterios más complejos.


## Análisis Dinámico

Una vez finalizado el análisis estático del binario, procedo a reabrirlo en modo debugger para realizar un análisis dinámico. En esta etapa, pasaré unos datos de prueba y mediante el uso de breakpoints compruebo la ejecución del programa.

![img](/imgs/write-ups/reversing/keygen/8.png#center)

Situo un breakpoint en la linea **0x0804860a** que es donde se realiza la comparación de la longitud con el valor **8** y a continuación ejecuto el binario.

![img](/imgs/write-ups/reversing/keygen/9.png#center)

Se observa que la primera comprobación se supera correctamente. Esto ocurre porque la cadena introducida tiene una longitud de 10 caracteres, incluyendo el carácter de salto de línea (ENTER). El programa continúa su flujo hacia las siguientes validaciones.

![img](/imgs/write-ups/reversing/keygen/10.png#center)

En la validación se realiza la multiplicación de los datos en **var_8ch** y **var_8bh** para despues compararlo con **var_9ch**. Al analizar que se encuentra en estas posiciones de memoria descubro los dos primeros valores apuntan a los dos primeros caracteres del nombre introducido y la variable **var_9ch** apunta al primer valor de la clave introducida.

![img](/imgs/write-ups/reversing/keygen/11.png#center)

De aquí deducimos que el algoritmo realiza la multiplicacion entre dos caracteres consecutivos y despues realiza una comparación con el valor asociado de la clave.

Conocido el funcionamiento, comienzo con el desarrollo del generador de claves

## Key Generator

### Opción 1
Para crear el generador de claves, desarrollo un script en Python que toma como entrada el nombre de usuario. A partir de este nombre, aplico una lógica basada en la multiplicación de caracteres adyacentes. El producto resultante de estas operaciones se formatea y se imprime siguiendo el diseño esperado de la clave. 

```py
def procesar_cadena(cadena):
    if len(cadena) < 8:
        print("La cadena debe tener al menos 8 caracteres.")
        return

    productos = []
    for i in range(0, 8, 2):
        # Toma pares de caracteres
        if i + 1 < len(cadena):
            prod = ord(cadena[i]) * ord(cadena[i + 1])
            productos.append(f"{prod:05d}")
        else:
            # Si la longitud es impar, toma el último carácter con un valor neutro
            prod = ord(cadena[i]) * 1
            productos.append(f"{prod:04d}")

    # formato xxxx-xxxx-xxxx
    resultado = "-".join(productos)
    print(resultado)

cadena = input("Introduce una cadena de al menos 8 caracteres: ")
procesar_cadena(cadena)
```

![img](/imgs/write-ups/reversing/keygen/12.png#center)


### Opción 2

Otra alternativa consiste en invertir el proceso: introducir un valor de clave específico y calcular el nombre de usuario correspondiente. Para ello, desarrollo un script que, a partir de la clave deseada, descompone los valores y determina los caracteres necesarios para reconstruir un nombre de usuario válido según la lógica del algoritmo analizado. Esto permite generar un nombre de usuario compatible.

```py
def encontrar_parejas(numero):
    if numero <= 0:
        print("El número debe ser un entero positivo.")
        return 0  # Devolver 0 si el número no es válido
    
    parejas = []
    for i in range(1, int(numero**0.5) + 1):  # Solo iteramos hasta la raíz cuadrada
        if numero % i == 0:
            pareja = (i, numero // i)
            # Verificar si ambos números están en el rango de caracteres imprimibles
            if 33 <= pareja[0] <= 125 and 33 <= pareja[1] <= 125:
                parejas.append(pareja)
    
    print(f"Parejas de números que multiplicados dan {numero}:")
    if parejas:  # Si se encontraron parejas
        for p in parejas:
            # Construir la salida con los caracteres correspondientes
            print(f"{p} -> {chr(p[0])}{chr(p[1])}")
        return 1  # Indicar que se encontró al menos una pareja
    else:
        print("No se han encontrado caracteres para ese número.")
        return 0  # Indicar que no se encontró ninguna pareja


# Ciclo para solicitar números al usuario
counter = 0
while counter < 4:
    try:
        numero = int(input(f"{counter} -- Introduce un número entero positivo: "))
        counter += encontrar_parejas(numero)
    except ValueError:
        print("Por favor, introduce un número entero válido.")

```

![img](/imgs/write-ups/reversing/keygen/13.png#center)

Este script es un poco más complicado ya que puede haber más de una combinacion de caracteres que al ser multiplicado den un mismo número o puede no haber ninguna.

:) 
