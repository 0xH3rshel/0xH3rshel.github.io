---
layout: post
title: Crossbow - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Cromiphi  
**Dificultad**: Medio

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow.png)

## Reconocimiento

Comienzo realizando una enumeración de los puertos abiertos en la máquina.

```bash
$ sudo nmap -p- crossbow.hmv
Starting Nmap 7.94SVN ( https://nmap.org ) at 2023-12-23 11:00 CET
Nmap scan report for crossbow.hmv (192.168.1.27)
Host is up (0.00032s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
9090/tcp open  zeus-admin
```

No hay que prestar mucha atención a "zeus-admin", eso lo indica **nmap** porque no reconoce el servicio que se está ejecutando. Solo nos interesa que el puerto 9090 está abierto.

### Puerto 80

En el puerto 80 encontramos un servidor http típico. Muestra información sobre **Polo**, sus hobbies y otros datos, pero lo relevante son los archivos JavaScript que podemos encontrar si abrimos el depurador del navegador.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_1.png)

Como se puede ver, hay dos archivos interesantes: **app.js** y **config.js**.

En el archivo **config.js**, encontramos una variable llamada HASH-API-KEY que podemos decodificar en este sitio web: **[Decode snefru](https://md5hashing.net/hash/snefru)**

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_2.png)

Guardamos el resultado y continuamos con la enumeración.

### Puerto 9090

Continuando la enumeración en el puerto 9090, nos encontramos con un formulario de inicio de sesión para una máquina Debian.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_3.png)

Pruebo con el usuario **Polo** y la contraseña obtenida del hash, ¡y estamos dentro!

En la parte izquierda, hay acceso a una terminal con la que podemos ejecutar comandos como Polo.

## Escalado de Privilegios

Esta parte es muy interesante, y voy a intentar explicarlo lo mejor posible.

**Nota**
- Si ejecutamos **linpeas.sh** o realizamos una enumeración manual del servidor, nos damos cuenta de que estamos dentro de un contenedor **Docker**. Esto será muy importante más adelante.

### SSH-Agent

Al listar los procesos en ejecución, vemos que el usuario **lea** tiene un "agent" en ejecución con el PID 1087.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_4.png)

### SSH-Agent-Hijacking

Nuestro objetivo será realizar un SSH-Agent-Hijacking. En el siguiente enlace se muestran los conceptos teóricos del ataque: [SSH-Agent-Hijacking](https://n3dx0o.medium.com/understanding-ssh-agent-and-ssh-agent-hijacking-a-real-life-scenario-2522475f7d8e), y en Hacktricks encontramos cómo explotarlo también: [Hacktricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/ssh-forward-agent-exploitation)

Si continuamos con la enumeración, vemos que el agente se encuentra en el directorio "/tmp", pero no podemos leerlo.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_5.png)

## Generación SSH-Agent

Si creamos un nuevo agente como el usuario Polo, veremos que el agente generado en la carpeta dentro de /tmp tiene un valor muy similar al PID del proceso que lo ejecuta.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_6.png)

Por lo que, conociendo el PID del agente del usuario **lea**, podemos realizar fuerza bruta y adivinar el nombre de su archivo agente.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_7.png)

Mmmmmm... Esto no ha funcionado.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_8.png)

También podemos ver que existe otro usuario llamado **Pedro**. Ahora intentaré de nuevo con ese usuario y conectándome a la máquina real; recordemos que estamos dentro de un contenedor Docker.

Sustituyo el nombre de la máquina, en lugar de **crossbow**, paso a utilizar la dirección IP (NO LA DEL CONTENEDOR).

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_9.png)

Mantengo pulsada la tecla ENTER mientras prueba con los distintos valores hasta encontrar el correcto.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_10.png)

¡Ha conseguido conectarse!

## Root

Enumerando los puertos abiertos, vemos que hay uno solo accesible desde localhost.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_11.png)

Para poder acceder desde mi máquina Kali, realizo una redirección de puertos con **socat**.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_12.png)

### Semaphore

Lo primero que encuentro es un formulario de inicio de sesión.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_13.png)

Esta parte no tiene dificultad, ya que probando credenciales rápidamente descubrimos que tanto el usuario como la contraseña son "admin".

Para conseguir acceso como usuario root, lo primero que hago es crear el siguiente "playbook" en el directorio /tmp.

![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_14.png)

A continuación, hay que seguir los siguientes pasos:

1. Añado un nuevo repositorio a Semaphore: /tmp
![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_15.png)

2. Creo un nuevo template, cuyo playbook sea el fichero shell.yml creado anteriormente.
![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_16.png)

3. Al ejecutarlo va a dar un error sobre "locales".
![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_17.png)

4. Para corregirlo, edito "dummy variable" añadiendo configuración de idioma.
![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_18.png)

5. Ahora ya todo funciona.
![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_19.png)

6. ¡Root!
![img](../imgs/write-ups/hackmyvm/crossbow/crossbow_20.png)

Muchas gracias a **Cromiphi** por esta máquina.