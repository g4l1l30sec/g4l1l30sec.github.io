---
description: >-
  G4l1l30
title: Frustrando al Actor Malicioso - Parte II
date: 2022-08-1 11:40:00 -0600
categories: [MISC]
tags: 
- PurpleTeam
- BlueTeam  
- RedTeam
- MeetUP  # TAG names should always be lowercase
show_image_post: true
image: /assets/img/Linux/ScriptKiddie/InfoCard-ScriptKiddie.png
layout: post
category: blog
author: RobertEncarnacion
---



# <center>Frustrando al Actor Malicioso - Parte II</center>



### <center>Payload Delivery and Execution desde una perspectiva Blue Team </center>



### SMB Relaying o la papa caliente del Actor Malicioso



<center><img src="https://sensorstechforum.com/wp-content/uploads/2020/03/CVE-2020-0796-smbghost-vulnerability-sensorstechforum.jpg" title="" alt="CVE-2020-0796: Ahora me ves, Now You Don't" data-align="center"></center>

En esencia, los protocolos mas viejos no fueron concebidos para ser seguros,**HTTP** no fue concebido para ser seguro, en **FTP** solo importa transferir archivos, Telnet solo desea que te conectes al servidor, **DNS** a solo le importa que encuentres a tu vecino en un gran directorio de llamadas, asi podemos enumerar unos cuantos mas, pero sin duda alguna, un protocolo que ha sido una maldicion o bendicion para los ambientes Windows es **NT Lan Manager** AKA NTLM. 



Hablaremos sobre el delivery payload en forma de movimiento lateral NTLM Relay, que tambien sirve para lograr un domain admin(esta clasificado como uno de los metodos convencionales junto a Kerberoasting, AD Misconfiguration y AS-REP Roasting), y por que, a diferencia de estos, es viejo, dificil de matar y es usado hoy dia.



Este protocolo de autenticacion (o familia de protocolos de autenticacion), fue concebido en su momento como una solucion para que en la fase de autenticacion, el password no viaje en texto plano, pero este acarrea una serie de problemas, entre lo que destacan el no autenticarse en el server (esto lo veremos mas adelante). 



NTLM ha sido sustituido hace casi 2 decadas por Kerberos, pero aun esta vivo, aun persiste y quizas se quede otra decada mas (?).



Antes de seguir, aclaremos algunas terminologias para evitar confusiones.



1. **NT Hash y LM Hash** son versiones codificadas de contraseñas de usuario. Los hashes de LM están totalmente obsoletos y no se mencionarán en este artículo. El hash de NT se denomina comúnmente, erróneamente en mi opinión, "hash de NTLM". Esta designación es confusa con el nombre del protocolo, NTLM. Por lo tanto, cuando hablemos del hash de la contraseña del usuario, nos referiremos a él como hash NT.

2. **NTLM** es, por tanto, el nombre del **protocolo de autenticación**. También existe en la versión 2.

3. N**TLMv1 Response y  NTLMv2 Response** será la terminología utilizada para referirse a la respuesta de desafío(challenge) enviada por el cliente, para las versiones 1 y 2 del protocolo NTLM.

4. **Net-NTLMv1 y Net-NTLMv2** son pseudo-neo-terminologías que se utilizan cuando el hash NT se denomina hash NTLM para distinguir el hash NTLM del protocolo. 

5. **Net-NTLMv1 Hash y Net-NTLMv2** Hash también son terminologías para evitar confusiones, pero tampoco se utilizarán en este artículo.



No pretendo entrar en detalles y explicar el funcionamiento en si, usare ilustraciones para un rapido desarrollo, por que una imagen vale mas que mil palabras(...digamos que la flojera persiste)



De la siguiente forma funciona NTLM autenticacion :



<center><img title="" src="http://3.bp.blogspot.com/-IieWjCCfBFc/Ve0g4I8n6nI/AAAAAAAAANk/0m8XiWAYCqg/s400/IC50098.gif" alt="RANDIKA'S TECH BLAST: How NTLM authentication works?" data-align="center" width="608"></center>



Es muy sencillo, de entender, el problema radica en que, quien valida el Challenge es el DC, y en este paso, es donde una atacante se aprovecha, pues si el recurso que el cliente solicita no se encuentra, en el proceso llamado "Negotiate", el DC en primer instancia negocia por la autenticacion de Kerberos pero este, al no encontrar el Service Principal Name , procede a hacer un Downgrade a NTLM!

+Info sobre SPN : [Service Principal Names - Win32 apps | Microsoft Docs](https://docs.microsoft.com/en-us/windows/win32/ad/service-principal-names)

No es el servidor a donde se solicita el recurso quien hace el downgrade de protocolo, es el mismo DC quien lo hace, ven el problema?  Este radica, en que la devuelta (paso 5) , un atacante puede interceptar los paquetes y hacer un especie de "relay" entre Servidor - Cliente, ya que al final, quien valida es el DC, no el servidor.



Este problema se resuelve con Kerberos, debido a que el DC generaria un Ticket (TGT), el servidor tiene que ser capaz de resolver dicho ticket y luego proceder, por esto NTLM fue sustituido por Kerberos, ya que este ultimo valida el servidor.  https://answers.microsoft.com/en-us/msoffice/forum/all/ntlm-vs-kerberos/d8b139bf-6b5a-4a53-9a00-bb75d4e219eb



Asi que, NTLM es susceptible a un ataque llamado MiTM o Man in the Middle, que no es mas que un atacante en el medio, interceptando paquetes de forma pasiva/activa .

<center><img title="" src="https://en.hackndo.com/assets/uploads/2020/03/ntlm_relay_basic.png" alt="RANDIKA'S TECH BLAST: How NTLM authentication works?" data-align="center" width="608"></center>




En la ilustracion anterior, el atacante esta interceptando los paquetes entre el servidor y el cliente, ya que no hay un identificacion valida entre ambos. 



En este proceso de Negotiate - Challenge - Authenticate, hay una serie de informacion que entra en juego, y es que el cliente envia el Nombre del Dominio, Usuario y Password (en Hash NTLMv2) como parte del challenge



<center><img title="" src="https://en.hackndo.com/assets/uploads/2020/03/ntlm_relay_example_smb_smb_no_signing_pcap_response_2.png" alt="Relai NTLM - Response" width="702"></center>



Digamos que los Windows Server, por politica refuerzan la complejidad del password pero, que hay de un usuario local?  si este es interceptado en el Response, un actor malicioso puede facilmente `crackear` el password :). 



## Demo : Interceptando el Challenge-Response de NTLM



En este demo podemos ver como interceptar el challenge-response de NTLM utilizando Responder, este ultimo es una herramienta que envenena Netbios y LLMNR (es decir, se pone a la escucha, intercepta y reenvia paquetes en esos protocolos), a su vez la mitigacion del mismo.



[JSjdajAX!!@)$@FKRedTeamRD#(#)!S - YouTube](https://www.youtube.com/watch?v=wKKMERdrgX0)



(El nombre no es del todo original). Praticamente para la mitigacion hemos aplicado la GPO en nuestro DC : 



`Local Computer Policy > Computer Configuration > Administrative Templates > Network > DNS Client.` y habilitamos la politica `Turn OFF Multicast Name Resolution` 



### El peligro del Relay

Pero el problema radica en que esta informacion puede hacer un "relay" entre varios hosts en un Active Directory, aqui donde entra en juego el NTLM Relay, donde se necesitan minimo 2 Hosts y un usuario privilegiado para entrar en juego.



<center>![](C:\Users\robencarnacion\AppData\Roaming\marktext\images\2022-08-01-15-09-33-image.png)</center>



Esto es posible por dos cosas : 



Cuando un cliente se autentica en un servidor para hacer algo, debemos distinguir dos cosas:

**Autenticación,** que permite al servidor verificar que el cliente es quien dice ser.


**La sesión**, durante la cual el cliente podrá realizar acciones.


Así, si el cliente se ha autenticado correctamente, **podrá acceder a los recursos que ofrece el servidor**, como recursos compartidos de red, acceso a un directorio LDAP, un servidor HTTP o una base de datos SQL. Esta lista obviamente no es exhaustiva.



### Demo : NTLM Relay y mitigacion.



En este ejemplo tenemos una maquina Kali, 2 Host Windows y un DC. 



[Relay I - YouTube](https://www.youtube.com/watch?v=adLsXv5xBUU)



y ahora la mitigacion : 

[Relay II - YouTube](https://www.youtube.com/watch?v=IJd5ZhhsUuo)



Para la mitigacion hemos activado que los servicios de Red, siempre esten firmados, aqui entra en juego Message Integrate Code (MIC), y si, hay forma de hacerle bypass, especialmente NTLMv1 se puede hacer un Drop de MIC.



La politica en cuestion es ` Security Settings > Local Policies > Security Options > Microsoft Network Server > Digitally sign communications (always)` 

https://www.ultimatewindowssecurity.com/wiki/page.aspx?spid=MSNSalways





## Consideraciones para erradicar NTLM o al menos intentarlo y no morir.



<center><img src="https://areajugones.sport.es/wp-content/uploads/2019/02/Obsidian-Vault-Boy.jpg" title="" alt="Fallout contará con una figura Nendoroid de su Vault Boy" data-align="center"></center>



Es probable que si tienes aplicaciones de terceros en tu Active Directory, NTLM este presente, que si tienes aplicaciones legacy, NTLM este presente, NTLM es dificil de matar, y el mal trato hacia ese protocolo, puede arruinar el dia a cualquier equipo de Seguridad y nuestros queridos accionistas o CISO no estarian contento.



Vamos a enumerar un listado de CVE :



ProxyLogon **(CVE-2021-26855, CVE-2021-27065)** y ProxyShell **(CVE-2021-34473, CVE-2021-34523, CVE-2021-31207)** de Orange Tsai, PetitPotam **(VDB-179650)** de topotam y varias vulnerabilidades en servicios de certificados de Active Directory **(ADCS)** configurados de forma insegura por Will Schroeder y Lee Christensen.



Saben que tienen en comun? NTLM, esto por enumerar CVE recientes :).



Entre las medidas que podemos optar estan: 



1. **AV efectivo/Filtro de trafico de red** : No todos los AV funcionarian,  por otra parte un buen filtro de trafico de Red seria ideal, existen software comerciales como DarkTrace que son capaces de detectar anomalias.

2. **Politicas de grupo (Signing)** : Esto lo hemos visto en el demo de mitigacion, no es mas que activar el MIC aka Firma en los servicios de red, sin embargo es facil de burlar si usas NTLMv1, es alto recomendable deshabilitar NTLMv1 en produccion.

3. **EPA y Channel Binding** : Digamos que un cliente solicita un recurso SMB, pero un atacante desea un HTTP? eso es posible, y aqui nos ayuda EPA (Enhanced Protection for Authentication)



Una **4ta** opcion que podemos tener en cuenta es **Auditar NTLM** , existen politicas para esto, erradicar o disminuir el uso de NTLM no es cosa de un dia para otro, es dedicar recursos durante meses.



En nuestro **Group Policy Editor** tenemos : `Computer Configuration\Policies\Windows Settings\Security Settings\Local Policies\Security Options`



En el que tenemos las siguientes politicas:

```
| Setting                                                                   | Value                            |
| ------------------------------------------------------------------------- | -------------------------------- |
| Network security: Restrict NTLM: Audit Incoming NTLM Traffic              | Enable auditing for all accounts |
| Network security: Restrict NTLM: Audit NTLM authentication in this domain | Enable all                       |
| Network security: Restrict NTLM: Outgoing NTLM traffic to remote servers  | Audit all                        |
```


*Audit NTLM authentication in this domain* debe ser aplicada al Domain Controller, las otras dos a todos los sistemas. Podemos auditar los logs en  `Applications And Services Logs\Microsoft\Windows\NTLM\Operational`



Por otro lado podemos bloquear NTLM de la siguiente forma :



En nuestro **Group Policy Editor** tenemos : `Computer Configuration\Policies\Windows Settings\Security Settings\Local Policies\Security Options`

```

| Setting                                                                  | Value             |
| ------------------------------------------------------------------------ | ----------------- |
| Network security: Restrict NTLM: Incoming NTLM traffic                   | Deny all accounts |
| Network security: Restrict NTLM: NTLM authentication in this domain      | Deny all          |
| Network security: Restrict NTLM: Outgoing NTLM traffic to remote servers | Deny all          |
```

La politica `*NTLM authentication in this domain*  es aplicada al Domain Controller, las otras dos a todo los sistemas.



Podemos aplicar excepciones en : 



- **Network security: Restrict NTLM**: Add remote server exceptions for NTLM authentication
- **Network security: Restrict NTLM:** Add server exceptions in this domain



Adicional a todo esto, es bueno considerar los patchs 



1. Drop the MIC [ **CVE 2019-1040** ]

2. DCE/RPC Relay [ **CVE 2021-1678** ]

3. PetitPotam [**CVE 2021-36942**] , esto es un NTLM Relay con esteroides para obtener un golden ticket.



Y por ultimo y no menos importante, deshabilitar **Printer Spooler** en los servidores criticos.
