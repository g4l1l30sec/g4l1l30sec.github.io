---
description: >-
  G4l1l30 Memory Dump Analysis de WannaCry
title: MemoryDump WannaCry | Parte I
date: 2021-06-1 15:40:00 -0600
categories: [Malware Analysis]
tags: [Volatility, WannaCry, MemoryDump]     # TAG names should always be lowercase
show_image_post: true
image: /_posts/WannaCry/ransomware-wannacry.jpg
---
# Analisis de Memory de WannaCry | Parte I



![](/assets/img/WannaCry/ransomware-wannacry.jpg)



En esta entrada vamos a analizar un dump de memory de un host infectado con WannaCry usando **Volatility** , solo analizaremos el analisis forense de memoria, es decir, no se va a cubrir el vector iniciar, la propagacion y el recovery.



# WannaCry

**WannaCry**, también conocido como **WanaCrypt0r 2.0**,[1](https://es.wikipedia.org/wiki/WannaCry#cite_note-:0-1) es un [*programa dañino*](https://es.wikipedia.org/wiki/Malware) de tipo *[ransomware](https://es.wikipedia.org/wiki/Ransomware)*. En septiembre de 2018, el Departamento de Justicia de los Estados Unidos inculpó al norcoreano [Park Jin Hyok](https://es.wikipedia.org/wiki/Park_Jin_Hyok) de ser el creador de WannaCry y haber acometido el ataque informático de alcance mundial en 2017. + Info [WannaCry](https://es.wikipedia.org/wiki/WannaCry)

# Disclaimer

1. Favor usar una VM para este tipo de analisis.
2. Si necesitas el zip(junto con el password) puedes contactarme en Twitter o Telegram
3. No nos hacemos responsables sobre cualquier consecuencias/dano si no obedece lo descrito aqui.
4. Esto es con fines educativos.

# Analysis.

La metodologia a usar son los 6 pasos de SANS:

1. Identify rogue processes 
2. Analyze process DLLs and handles 
3. Review network artifacts 
4. Look for evidence of code injection 
5. Check for signs of rootkit
6. Dump suspicious processes and drivers 

+ +Info [SANS Memory Analysis.](https://www.sans.org/security-resources/posters/memory-forensics-cheat-sheet/365/download)

Antes de proceder, vamos a hacer un analisis estatico para verificar indicadores de compromisos, empezaremos usando el comando `strings` y `grep` : 

![](/assets/img/WannaCry/strings.png)

La unica URL conseguida y es lo que se conoce como **KillSwitch** .El investigador MalwareTech descubrió que los programadores del ransomware lo habían creado para comprobar si una URL sin sentido conducía a una página web activa. Curioso por qué el ransomware buscaría ese dominio, MalwareTech lo registró él mismo. Resulta que esa inversión de $ 10,69 fue suficiente para detener el ransom.Resultó que mientras el dominio no estuviera registrado e inactivo, la consulta no tuvo ningún efecto en la propagación del ransomware. Pero una vez que el ransomware verificó la URL y la encontró activa, se cerró. +Info : [How to Accidentally stop a Global Cyber Attack by MalwareTech](https://www.malwaretech.com/2017/05/how-to-accidentally-stop-a-global-cyber-attacks.html)

![](/assets/img/WannaCry/strings_exe.png)

Estos indicadores de compromiso de WannaCry le sirve para ejecutar tareas.

# Volatility

Volatility es framework open-source para Incident Response y Malware Analysis. Primero vamos a obtener informacion sobre el dump.

```bash
vol.py -f wcry.raw imageinfo
```

![vol_01](/assets/img/WannaCry/vol_01.png)

Ahora usaremos el plugin **pslist**  para verificar los procesos, es importante estar familarizado con los procesos nativos del sistema operativo:

```bash
vol.py -f wcry.raw pslist
```

![](/assets/img/WannaCry/vol_02.png)

**PID 1940** y  **PID 740** ambos procesos se ven completamente extraños, ahora usaremos **psscan** para ver un listado mas completo de los procesos (nos lista los procesos terminados, que seria **PIDD**):

```bash
vol.py -f wcry.raw --profile=WinXPSP3x86 psscan
```



![](/assets/img/WannaCry/vol_03.png)

**WannaDecryptor** y **tasks*** estan relacionados entre si, eso lo podemos observar por el **PIDD: 1940**, vamos analizar el timeline

![](/assets/img/WannaCry/vol_04.png)

Y ordenamos con **sort**

![](/assets/img/WannaCry/vol_05.png)

Podemos hacer una busqueda de inteligencia sobre esos procesos :) , ahora vamos a listar los **directorios** y **librerias** utilizando el plugin ``dlllist`` apuntando al **PID 1940 y 740**

```bash
vol.py -f wcry.raw --profile=WinXPSP3x86 dlllist -p 1940
```

![vol_06](/assets/img/WannaCry/vol_06.png)

Un directorio bastante sospechoso :) 

![vol_07](/assets/img/WannaCry/vol_07.png)

`WannaDecryptor` usa el mismo directorio, tambien podemos observar una las siguientes **dll** :

1. ADVAPI32.dll : Empleada para querys en el registro
2. Secur32.dll : Encriptacion
3. urlmon.dll : Interactuar con navegadores
4. WS2_32.dll : Creacion de sockets
5. WININT.dll : Interacion de red con aplicaciones

Ahora verificaremos el plugin **handles** para listar files, registry key, events, desktops, threads,etc.

```bash
vol.py -f wcry.raw --profile=WinXPSP3x86 handles -p 1940
```

![vol_08](/assets/img/WannaCry/vol_08.png)

Observamos lo que se conoce como Mutex; esto sirve para que una vez el host este infectado, de forma preventiva no corre mas de una instancia del malware, en este caso tenemos `MsWinZonesCacheCounterMutexA` (En google obtenemos mucha info del mismo).

Dado de que no encontramos conexiones con los plugins **connections** y **connscan** podemos usar `bulk_extractor` para extraer las conexiones desde el memory dump:

```bash
bulk_extractor -E net -o pcap/ wcry.raw
```

![vol_09](/assets/img/WannaCry/vol_09.png)

Y vamos a obtener unos cuantos IoC:

```bash
tshark -T fields -e ip.src -r packets.pcap 
```

![vol_10](/assets/img/WannaCry/vol_10.png)

Obtenemos los siguientes IPs :

```text
134.119.3.164
192.168.56.101
199.254.238.52
213.61.66.118
```

