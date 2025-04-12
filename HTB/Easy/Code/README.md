# Informe Explotación - Code

Este informe documenta el proceso de explotación de la máquina Code, parte de la plataforma Hack The Box (HTB). La máquina presenta diversas vulnerabilidades críticas que permitieron comprometer el sistema desde la ejecución remota de código (RCE) hasta la escalada de privilegios a nivel root. A través de un análisis detallado de los servicios expuestos, las configuraciones inseguras y las fallas de validación, se pudieron identificar múltiples vectores de ataque que fueron explotados para obtener acceso completo al sistema.

El objetivo de este informe es proporcionar un resumen detallado de cada paso realizado durante la explotación, desde la recopilación de información hasta la obtención de las flags de usuario y root.

## Índice

1. [Informe de Explotación - Code](#informe-de-explotación---code)
2. [Información General](#información-general)
3. [Reconocimiento](#reconocimiento)
    - [Escaneo de Puertos](#escaneo-de-puertos)
    - [Enumeración de Servicios](#enumeración-de-servicios)
4. [Análisis de Vulnerabilidades](#análisis-de-vulnerabilidades)
   - [1. Ejecución de Código Python (RCE)](#1-ejecución-de-código-python-rce)
   - [2. Posible vulnerabilidad SQLi](#2-posible-vulnerabilidad-sqli)
5. [Explotación Inicial](#explotación-inicial)
    - [Obtener los Hashes de Contraseña](#obtener-los-hashes-de-contraseña)
6. [Post-Explotación](#post-explotación)
    - [Escalada de Privilegios](#escalada-de-privilegios)
7. [Conclusiones](#conclusiones)


## Información General
- **Nombre de la máquina:** Code
- **Nivel de dificultad:** Fácil
- **Dirección IP:** 10.10.11.62
- **Sistema Operativo:** Linux

## Reconocimiento
### Escaneo de Puertos
Comando utilizado:
``` bash
nmap -sC -sV 10.10.11.62
```
Resultado:
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/nmap.png" alt="nmap" style="border-radius: 10px;">
</p>

<br>

- -sC: Activa los scripts para detectar vulnerabilidades comunes.
- -sV: Detecta las versiones de los servicios.

### Enumeración de Servicios
- SSH (Puerto 22): Servicio activo, pero sin credenciales conocidas.
- HTTP (Puerto 5000): Un servidor web con un editor de Python que permite ejecutar código, y donde también es posible registrarse y autenticarse. Esta configuración podría ser vulnerable a Remote Code Execution (RCE) si no se valida adecuadamente la entrada del usuario.


## Análisis de Vulnerabilidades
Durante la enumeración del servicio web en el puerto 5000, se detectaron varias funcionalidades que pueden ser vectores de ataque potenciales.

### 1. Ejecución de Código Python (RCE)
La interfaz principal muestra un editor que permite escribir y ejecutar código Python. Si el backend no aplica ningún tipo de restricción sobre las funciones disponibles, es posible ejecutar código arbitrario en el sistema.

Funciones potencialmente explotables: os.system(), subprocess, lectura/escritura de archivos, etc.

### 2. Posible Vulnerabilidad SQLi
Los formularios de inicio de sesión y registro podrían ser vulnerables a inyecciones SQL si no se filtran correctamente las entradas. Se realizaron pruebas básicas con ' OR '1'='1 y valores especiales para observar el comportamiento del servidor.

## Explotación Inicial
Se llevaron a cabo diversas pruebas para verificar si el código se ejecutaba directamente en el servidor. Inicialmente, cualquier intento de importar módulos del sistema como os, subprocess, o incluso funciones como __import__() era rechazado por la plataforma, devolviendo errores de seguridad.

Aunque los intentos iniciales de importar módulos fueron bloqueados, se identificó que el sistema permitía la ejecución de código mediante la función print().

Se realizaron pruebas para acceder a objetos internos, como:

``` python
print(open('/etc/passwd').read())
print(os.environ['PATH'])
```

Finalmente, se construyó el siguiente payload:
``` python
print([(user.id, user.username, user.password) for user in User.query.all()])
```

Este código realiza una consulta a la base de datos utilizando el objeto User, que representa la tabla de usuarios. El método User.query.all() recupera todos los usuarios de la base de datos. Luego, genera una lista con la información de cada usuario:
- user.id: Identificador del usuario.
- user.username: El nombre de usuario.
- user.password: La contraseña en formato hash.

Resultado:
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/query.png" alt="query" style="border-radius: 10px;">
</p>

<br>

### Obtener los Hashes de Contraseña
A través de la explotación del código Python, se obtuvieron los siguientes hashes de contraseñas asociados a dos usuarios:

- **Usuario 1:** development
- **Hash:** 759b74ce43947f5f4c91aeddc3e5bad3

- **Usuario 2:** martin
- **Hash:** 3de6f30c4a09c27fc71932bfc68474be

Para descifrar los hashes, se utilizó el servicio CrackStation, una herramienta rápida para la recuperación de contraseñas mediante el uso de tablas de búsqueda (rainbow tables).
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/hashes.png" alt="hashes" style="border-radius: 10px;">
</p>

<br>

## Post-Explotación
Una vez obtenidas las contraseñas en texto plano de los usuarios, se puede intentar establecer una conexión SSH con la máquina utilizando las siguientes credenciales.

Al intentar conectarse con el usuario **development**, se deniega el acceso por falta de permisos.

Sin embargo, se logró establecer una conexión SSH con el usuario **martin**:
``` bash
ssh martin@10.10.11.62
```

La contraseña utilizada para la conexión fue: **nafeelswordsmaster**. Este paso permitió acceder al sistema como el usuario martin.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/ssh.png" alt="ssh" style="border-radius: 10px;">
</p>

<br>

Una vez dentro del sistema, se continuó con la exploración para identificar posibles escenarios de escalada de privilegios y realizar otras actividades de post-explotación.

### Escalada de Privilegios
Tras comprobar los permisos con el comando **sudo -l**, se identificó que el usuario **martin** tenía permiso para ejecutar el script **/usr/bin/backy.sh** con privilegios de administrador, sin necesidad de contraseña.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/sudo-l.png" alt="sudo-l" style="border-radius: 10px;">
</p>

<br>

Esta configuración representó un punto clave para explotar una posible escalada de privilegios, ya que permitía ejecutar el script **backy.sh** con permisos de root.

Se examinó el contenido del script con:
``` bash
cat /usr/bin/backy.sh
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/cat-backy.png" alt="cat-backy" style="border-radius: 10px;">
</p>

<br>

Observaciones clave:
- El script permite realizar copias de seguridad de ciertos archivos específicos a través de un archivo JSON **(task.json)**.
- Se ejecuta con privilegios de administrador, lo que le da acceso a archivos restringidos del sistema.
- Aunque el script solo permite copiar archivos dentro de determinadas carpetas **(/var/ y /home/)**, es posible manipular la entrada para intentar incluir archivos sensibles.

Para empezar, se modificó el archivo task.json para intentar realizar una copia de seguridad de algunas de las carpetas encontradas con el usuario martin, pero a las cuales no se tenía acceso.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/no-acces.png" alt="no-acces" style="border-radius: 10px;">
</p>

<br>

Código de **task.json**
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/codi-task1.png" alt="codigo-task1" style="border-radius: 10px;">
</p>

<br>

Una vez modificado el archivo JSON, se ejecutó el script con el siguiente comando:
``` bash
sudo /usr/bin/backy.sh task.json
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/ejecucion-script.png" alt="ejecucion-script" style="border-radius: 10px;">
</p>

<br>

El script se completó correctamente, creando un archivo de copia de seguridad con el nombre: **code_home_app-production_2025_April.tar.bz2**

Para extraer el archivo, se utilizó el siguiente comando:
``` bash
tar -xjf code_home_app-production_user.t_2025_March.tar.bz2
```

<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/tar.png" alt="tar" style="border-radius: 10px;">
</p>

<br>

Al navegar al directorio **/home/app-production**, se obtuvo la bandera del usuario.
- **Flag user:** 5c89db216f724a2a9a3a329ecc0f3d21
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/flag-user.png" alt="flag-user" style="border-radius: 10px;">
</p>

<br>

Con el éxito de la primera explotación, se intentó obtener la flag del usuario root. Para ello, se modificó nuevamente el archivo **task.json** para incluir este directorio en las copias de seguridad.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/codi-task2.png" alt="codigo-task2" style="border-radius: 10px;">
</p>

<br>

Tras modificar el archivo JSON, se ejecutó nuevamente el script con el mismo comando.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/ejecucion-script2.png" alt="ejecucion-script2" style="border-radius: 10px;">
</p>

<br>

En este caso, el script realizó con éxito una copia de seguridad del archivo root.txt. Se extrajo el archivo de la copia de seguridad y se obtuvo la flag de root.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/flag-root.png" alt="flag-root" style="border-radius: 10px;">
</p>

<br>

## Conclusiones
La explotación de la máquina **Code** ha expuesto varias vulnerabilidades críticas que han permitido comprometer completamente el sistema.

En primer lugar, la presencia de un editor de código Python accesible desde una interfaz web, sin medidas de seguridad adecuadas, abrió la puerta a una vulnerabilidad de *Remote Code Execution (RCE)*. Esta vulnerabilidad fue explotada para acceder a datos críticos, como las credenciales de usuario almacenadas en la base de datos.

Posteriormente, gracias a estas credenciales, se logró obtener acceso SSH a la máquina con el usuario martin, lo que permitió avanzar a la fase de post-explotación.

La falta de restricciones adecuadas en el script con permisos de administrador **(/usr/bin/backy.sh)** facilitó la escalada de privilegios. Aunque el script estaba diseñado para trabajar solo con directorios específicos como **/var/** y **/home/**, fue posible manipular las rutas para acceder a archivos del directorio **/root/** mediante una ruta relativa **(/var/....//root/)**. Este bypass permitió obtener la flag de root.

Este conjunto de vulnerabilidades pone de manifiesto la importancia de validar de forma estricta cualquier entrada del usuario, así como limitar las operaciones privilegiadas a los archivos estrictamente necesarios, evitando cualquier forma de manipulación de rutas o ejecución de código arbitrario.

<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/HTB/Easy/Code/media/img/result.png" alt="result" style="border-radius: 10px;">
</p>

<br>
