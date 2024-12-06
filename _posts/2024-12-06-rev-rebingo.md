---
layout: post
title: ReBingo Lab - Writeup
tags: [rev, radare, asm]
---

Autor: 0xH3rshel

# Introducción

En este post, voy a resolver el laboratorio **ReBingo** del libro X86 Software Reverse Engineering: Cracking and Counter-Measures. Este ejercicio es una excelente práctica para comenzar en la ingeniería inversa y aprender a identificar estructuras de control de flujo en ensamblador y analizar la lógica de un programa.

![img](/imgs/write-ups/reversing/rebingo/portada.jpg#center)

Este y el resto de laboratorios los podéis encontrar en el repositorio de [GITHUB](https://github.com/DazzleCatDuo/X86-SOFTWARE-REVERSE-ENGINEERING-CRACKING-AND-COUNTER-MEASURES).

Este laboratorio está compuesto por 8 binarios en los que habrá que identificar que estructura de control está siendo utilizada, si el binario contiene información de depuración o no (striped) y por último distinguir si el binario ha sido optimizado o no durante la compilación.

## Desafío 1

Comienzo abriendo el binario con la herramienta **radare2** y realizo un análisis inicial usando el comando **aaaa**.

El primer paso es determinar si el programa contiene información de depuración o no. Este análisis lo realizaré únicamente en el primer y segundo programa para mostrar la diferencia, ya que es fácil de identificar.

Para ello, utilizo el comando **iI**, que muestra información detallada sobre el binario. En las últimas líneas del resultado se puede observar que la información simbólica no ha sido eliminada, ya que el campo "stripped" aparece como falso.

![img](/imgs/write-ups/reversing/rebingo/1_1.png#center)

Esto lo podemos corroborar al listar las funciones del programa mediante "afl".

![img](/imgs/write-ups/reversing/rebingo/1_2.png#center)

Se puede ver que el nombre original de la función se ha mantenido. El cambio será evidente al comparar con el siguiente desafío.

Para continuar, me situo en la función que hay que analizar mediante el comando **s sym.key_check** e imprimo la función.

![img](/imgs/write-ups/reversing/rebingo/1_3.png#center)

La función verifica si el valor de la variable local **var_4h** es menor o igual a 10 (0xa) mediante la operación de comparación **cmp**. Si lo es, realiza un salto mediante **jle** (Jump if Less or Equal) al final de la función y termina sin realizar ninguna acción adicional. Si el valor es mayor a 10, llama a otra función (entry0) antes de finalizar.

Este caso es una comparación simple que se podría traducir a C mediante una sentencia **if**:

```c
if (var_4h > 0xa) {
    entry0();
}
```

Queda por determinar si se ha optimizado la compilación del programa, pero no hay indicios de ello por lo que descarto.

## Desafío 2

Al igual que en el desafío anterior, comienzo realizando un análisis completo del binario mediante **aaaa** y muestro la infomación con **iI**

En este caso podemos ver que si ha sido eliminada la información de depuración del binario "Stripped True".
![img](/imgs/write-ups/reversing/rebingo/2_1.png#center)

Al listar las funciones del programa es más evidente ya que el nombre de las funciones ha sido eliminado y solo se muestra un número identificativo.

![img](/imgs/write-ups/reversing/rebingo/2_2.png#center)

Me desplazo a la función **fcn.080480e0** y comienzo su análisis.

![img](/imgs/write-ups/reversing/rebingo/2_3.png#center)

La función inicializa una variable local con 1 y verifica si su valor es mayor que 9 mediante la instrucción **cmp**. Si lo es, realiza una serie de llamadas a **entry0**. Si no lo es, realiza una única llamada a **entry0** y salta directamente al final de la función. Los saltos condicionales jg (Jump if Greater) y el incondicional jmp (Jump) controlan este flujo lógico.

Dado que se realiza una comparación y ejecuta un codigo u otro dependiendo del resultado, podemos concluir que se trata de una sentencia **if-else**.

```c
if (var_4h > 9) {
    entry0();
    entry0();
}else{
    entry0();
}
```

Este desafío tampoco presenta indicios de haber sido optimizado.

## Desafío 3

A partir de este desafío no muestro el proceso para comprobar si se ha eliminado la información de depuración o no ya que se realiza de la misma manera que en los apartados anteriores.

![img](/imgs/write-ups/reversing/rebingo/3_1.png#center)

La función comienza inicializando la variable local **var_4h** con el valor 0 y realiza un salto incondicional a la dirección 0x80480f8. Desde este punto, entra en un bucle donde primero llama a la función entry0 y luego incrementa en 1 el valor de **var_4h**. Este bucle se repite mientras **var_4h** sea menor o igual a 9. Una vez que se supera este valor, la función finaliza.

Este comportamiento lo podemos asociar a la sentecia **for**.

```c
for (int var_4h = 0; var_4h <= 9; var_4h++) {
        entry0();
}
```

En este desafío tampoco hay indicios de optimizaciones.

## Desafío 4

Muestro la función y comienzo el análisis.

![img](/imgs/write-ups/reversing/rebingo/4_1.png#center)

En este caso nos encontramos otra vez con un bucle pero un poco distinto al anterior. La variable que se utiliza como contador ya no se almacena en una dirección de memoria si no que en el registro **ebx** el cual se irá reduciendo en 1 mediante la instrucción **sub ebx, 1** hasta que sea 0 y se finalice la función.

El equivalente en C sería el siguiente:

```c
for (int ebx = 10; ebx > 0; ebx--) {
        0x80480f0();
}
```

Pese a tratarse de la misma sentencia que el desafío anterior, tiene una estructura diferente. Esto se debe a que el binario ha sido optimizado durante la compilación lo cual podemos deducir de:

- El uso eficiente de registros para evitar accesos repetitivos a la memoria, que tiene un tiempo de acceso superior.
- Se han reducido al minimo las instrucciones en el prologo y epilogo de la función.
- Aparición de la instrucción NOP (No OPeration), probablemente para alinear de forma eficiente el tamaño de las instrucciones.

## Desafío 5

Tras analizar el binario número 5 me encuentro con la siguiente función:

![img](/imgs/write-ups/reversing/rebingo/5_1.png#center)

En este caso no encontramos ante otro bucle pero distinto de los dos anteriores ya que tras inicializar la variable que se utilizará como contador **mov var_4h, 1**, se ejecuta el cuerpo del bucle y por último la comparación. Por ello esta sentencia es un **do-while** en lugar de un bucle **for**.

Para aclarar, la diferencia principal entre un bucle for y un do-while radica en el momento de evaluar la condición: el **for** la evalúa ANTES de ejecutar el cuerpo del bucle, por lo que puede no ejecutarse si la condición no se cumple desde el inicio, mientras que el **do-while** evalúa la condición DESPUÉS de ejecutar el cuerpo, garantizando que el código se ejecute al menos una vez.

```c
int var_4h = 1;
do {
        entry0();
        var_4h++;
} while (var_4h <= 9);
```

Además, en este código no hay indicios de que haya sido optimizado.

## Desafío 6

Siguiente desafío...

![img](/imgs/write-ups/reversing/rebingo/6_1.png#center)

Aunque a primera vista este desafío pueda parecer más complejo, rápidamente se puede ver que no realiza más que instrucciones **cmp** y saltos condicionales **je** (Jump if Equal) e incondicionales **jmp**.

El programa comienza inicializando una variable local **var_4h** a 1 y después compara si su valor es igual a 1 y, si es verdadero, salta a una dirección que realiza una llamada a la función entry0. Luego, verifica si el valor de la variable es igual a 10 y, en ese caso, realiza otra acción específica con otra llamada a entry0 y por último comprueba si su valor es 0. En caso de no cumplirse ninguna de estas condiciones, realiza un salto incondicional al final de la función.

La descripción de esta función encaja perfectamente con la de una sentencia **switch**.

```c
int var_4h = 1; 
    switch (var_4h) {
        case 1:
            entry0();
            entry0();
            break;

        case 10:
            entry0();
            entry0();
            entry0();
            break;

        case 0:
            entry0();
            break;

        default:
            break;
    }
```

Aunque aparece la instrucción NOP, el código no parece estar optimizado debido al uso redundante de accesos a memoria y saltos innecesarios, los cuales podrían haberse eliminado o simplificado para reducir tanto la longitud como la complejidad del código.

## Desafío 7

Penúltimo programa :)

![img](/imgs/write-ups/reversing/rebingo/7_1.png#center)

Rápido, sencillo, funcional... Un código que pronto nos damos cuenta que está optimizado debido a la ausencia del prólogo y epílogo de la función y la aparición de la instrucción NOP para alinear la longitud del programa. Parece ser un bucle **for** que, debido al reducido número de iteraciones, el compilador ha optado por desenrollar, llamando a las funciones directamente en lugar de mantener la estructura del bucle.

## Desafío 8

Por último pero no menos importante...

![img](/imgs/write-ups/reversing/rebingo/8_1.png#center)

La función es una estructura if-else. Según si el valor de la variable var_ch (inicializada con 1) es menor o igual a 9 o no, ejecuta una llamada a sym.do_stuff en caso de ser menor que 9 y en caso contrario la ejecutará dos veces.

El código vemos que está optimizado debido a la ausencia de prologo de la función.

# Conclusión

Este laboratorio ha sido muy interesante para aprender sobre las distintas estructuras de control de flujo e identificarlas en un programa compilado con y sin optimizaciones. Analizar estos binarios ayuda a consolidar conceptos clave de ingeniería inversa y prácticas de optimización de código.

Queda así resuelta la primera parte del laboratorio e incito al lector a probar con la segunda que está disponible en el repositorio de [GITHUB](https://github.com/DazzleCatDuo/X86-SOFTWARE-REVERSE-ENGINEERING-CRACKING-AND-COUNTER-MEASURES).

:)
