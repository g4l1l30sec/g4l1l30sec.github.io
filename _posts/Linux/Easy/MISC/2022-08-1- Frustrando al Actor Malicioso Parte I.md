---
description: >-
  G4l1l30
title: Frustrando al Actor Malicioso - Parte I
date: 2022-08-1 10:40:00 -0600
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


# <center>Frustrando al Actor Malicioso</center>

### <center>Payload Delivery and Execution desde una perspectiva Blue Team</center>

# 

# <center> Introduccion</center>



Los atacantes y los equipos de RedTeam no descansan, los equipos de BlueTeam/PurpleTeam o en su defecto, los equipos de Seguridad deben optar por estar a la par a sus adversarios. 



Es necesario aplicar la paradoja de la Reina Roja en nuestro dia a dia, hay que evolucionar constantemente para evolucionar. Esta entrada del blog quizas suene como un tema de Cyber Deception, pero no, nos limitaremos a aplicar soluciones preventivas en casos comunes donde un adversario puede atacarnos y sacar provecho de un ambiente desprotegido, a sinceridad, serian varias entradas que ire sacando a medida que tenga un chance.





<img title="" src="https://almargenoaxaca.files.wordpress.com/2020/04/alice_queen2-1665x1052-1.jpg?w=825&h=510&crop=1" alt="Paradoja de la Reina Roja" data-align="center">

— Bueno, en mi país —dijo Alicia aún jadeando—, si corres tan rápido durante tanto tiempo, sueles llegar a otro sitio…

— ¡Un país bastante lento! —replicó la Reina—.



## <center> Delivery payload o como no realizar la debida deligencia puede ser un estrago para tu ambiente.</center>



<img title="" src="https://i.ibb.co/BNVgyPC/2022-08-01-10-38-58-image.png" alt="punch" data-align="center">



Una vez que se hayan completado las actividades de reconocimiento, el adversario intentará entregar y ejecutar un payload, los metodos tipicos de intrusion usados hoy dia se encuentran : 



•**Email con attachments** malicioso o web pages(watering holes) a traves de (spear) phising. Ver  [How the biggest cyber heist in history was foiled | NordVPN](https://nordvpn.com/blog/the-lazarus-heist/)

•**Abusando de una falla en el perimetro externo** (a nivel aplicativo o infraestructura).  Ver [Eternal Blue](https://www.loginradius.com/blog/engineering/eternal-blue-retrospective/)

**•Insertando memoria usb**(autorun o de forma automática con Rubber Ducky) infectadas(para esto, se necesita interaccion con el target) . Ver [I Had My Mom Break into a Prison. Then, We Had Pie. - YouTube](https://www.youtube.com/watch?v=yqOGuXcLdOA)

**•Comprometer third party en el supply chain** y abusar de la confianza. Ver [Warnings (&amp; Lessons) of the 2013 Target Data Breach](https://redriver.com/security/target-data-breach)



Con base en diferentes publicaciones de los últimos años, podemos echar un vistazo a  las siguientes estadísticas interesantes que son relevantes para nosotros:



1. 82% de los data breach, son cometidos por el factor Humano, en el cual se incluye Social Attacks. [2022 Data Breach Investigations Report | Verizon](https://www.verizon.com/business/resources/reports/dbir/)

2. 66% Alrededor de dos tercios de las infracciones involucraron phishing, credenciales robadas y/o ransomware. [2022 Data Breach Investigations Report | Verizon](https://www.verizon.com/business/resources/reports/dbir/)

3. Es probable que casi el 20% de todos los empleados hagan clic en enlaces de correo electrónico de phishing. [Gone Phishing Tournament Report 2020 | Terranova Security](https://terranovasecurity.com/2020-gpt-report/?utm_campaign=En_GPTReport2020&utm_medium=Google&utm_source=Ads&utm_content=NewAd3&gclid=CjwKCAjw6fCCBhBNEiwAem5SO8oIgjFVtVzMA5pg-uSkRAho6S356pspA4bY3FBFk9FXCKW0Ksq-ExoCsHEQAvD_BwE)

4. Un Informe de amenazas reciente de ESET encontró que, en el tercer trimestre de 2020, los tipos más comunes de archivos maliciosos adjuntos a los correos electrónicos de phishing fueron los siguientes: Windows executables (74%) , Script files (11%), Office documents (5%).  https://www.welivesecurity.com/wp-content/uploads/2020/10/ESET_Threat_Report_Q32020.pdf



Estas estadísticas están en línea con lo que estamos viendo en la industria: El phishing es efectivo y fácil de hacer, lo que lo convierte en el arma preferida para la entrega de los Payload.



### <center>Metodos convencionales para prevenir el Payload Delivery</center>



![](https://media-exp1.licdn.com/dms/image/C4D12AQHaWjvpMywuJQ/article-cover_image-shrink_423_752/0/1520089499299?e=1665014400&v=beta&t=HoIhAlZI7FgpBUS4JbLhgcmAGUitSX4j7qtka3uYkhA)



Dados los métodos de delivery enumerados anteriormente, los siguientes son controles clave que pueden ayudar a evitarlos: 



1. Fortalecer la infraestructura de correo electrónico para bloquear los archivos adjuntos maliciosos entrantes o los correos electrónicos de phishing.

2. Reforzar la infraestructura web fortaleciendo los navegadores y configurando servidores proxy y DNS de forma segura.

3. Implemente políticas sólidas de seguridad de medios extraíbles.

4. Inviertir en la concientización sobre la seguridad del usuario final para educar a las personas sobre las estrategias típicas de los actores maliciosos.

5. Implementar una segmentación de red adecuada (tanto redes internas como de socios/terceros/accionistas).

6. Garantizar que se implementen los controles físicos adecuados para proteger la infraestructura corporativa.



Este trabajo no es de un solo departamento, es toda una directiva de proteccion integral en el que debe haber una sinergia entre diferentes gerencias (Seguridad de Sistemas, Seguridad de la Informacion, Seguridad fisica, etc). La seguridad no es cosa de un solo departamento y mucho menos de una sola persona(aun que si es sabido que la responsabilidad final recae en el CISO, ya que dicha posicion no es solo "una posicion" si no, mas bien un role en la organizacion).



## <center>La importancia de la educacion del usuario final</center>


<img src="https://user-images.githubusercontent.com/70044502/182217203-2ce16c00-b4b2-4385-9a91-b5e61afa93ae.png" title="" alt="" data-align="center">

Hay una razón por la que el usuario final suele llamarse el eslabón más débil de la seguridad . Incluso utilizando los mecanismos de defensa tecnológicos más avanzados, nuestra su organización aún puede ser violada por el error de un solo usuario.

El peligro no solo radica en los usuarios como receptores de correos electrónicos maliciosos, sino también en los usuarios como fuente de malas prácticas de seguridad. 

A menudo, las aplicaciones se utilizan a través de cuentas compartidas por varios usuarios. Para mantener las cosas "fáciles", estas cuentas tienen contraseñas débiles que todos pueden recordar.

Finalmente, los usuarios también representan una amenaza física. La tendencia BYOD actual introduce varios dispositivos móviles en las instalaciones de la organización.



En resumidas cuentas, podemos decir que :



1. La concienciación de los usuarios es un elemento clave para proteger nuestra empresa. El actor malicioso intentará obtener acceso a nuestro entorno abusando del enlace más débil, que normalmente son los usuarios finales.

2. Se debe educar a los usuarios para que comprendan cómo son los ataques y el papel que desempeñan en la prevención/detección de los mismos. Esto es importante para el personal, desde secretarias/asistentes hasta administradores de TI y ejecutivos de nivel alto.

3. La concientización sobre la seguridad del usuario final no es algo único; debe ser un proceso interactivo en el que los empleados reciban continuamente capacitación y educación personalizadas sobre ataques y sus consecuencias.



## <center>Paletas de colores para ejercicios de Phishing.</center>

<img title="" src="https://images.getdunmag.com/posts/475/image-7609d3.JPG" alt="Women and Fly Fishing Through the Ages: A More Complete Picture Emerges ~  DUN Magazine" width="292" data-align="center">

Hoy dia, poco ha cambiado entre las soluciones a la hora de implementar controles, tenemos herramientas Opensource y comerciales, estas ultimas a veces rodeadas de un Halo de misterio debido a la no entrega de un trial, pero no entraremos en detalles.



<img title="" src="https://i0.wp.com/derechodelared.com/wp-content/uploads/2018/04/gophish-1.png?fit=1200%2C675&ssl=1" alt="Gophish, la herramienta para entrenar usuarios contra el Phishing." width="544" data-align="center">

Del lado OpenSource tenemos uno bastante famoso que es Gophish, el cual provee un GUI administrativo version Web, donde las campanas son facilmentes creadas, ejecutadas y administradas. https://getgophish.com/



Si bien este ultimo nos obliga a registrar un dominio, aqui juega un poco la inteligencia del atacante y la inocencia del usuario final, podemos aplicar typosquatting, es decir, un usuario final seria capaz de diferenciar entre lo siguiente ?



1. banco.do

2. bancoo.do [un error de tipeo que puede aprovechar un atacante]

3. bаnco.do (?) acaso este ultimo es el 1ro? 



Si le echamos un vistazo , la "a" capital es similar a la "а" cyrilica, esto, sin embargo via correo no se distingue la diferencia, por suerte en nuestros navegadores son capaces de resolver estos nombres a lo que recae el formato "Punny Code", para el ultimo ejemplo, en nuestro navegador seria : 


<img title="" src="https://user-images.githubusercontent.com/70044502/182217831-09902bb5-9620-46b0-995d-fd8b1b56c574.png" alt="Gophish, la herramienta para entrenar usuarios contra el Phishing." width="544" data-align="center">



Seamos honesto, si el phishing es tan exitoso, no es por que el usuario se fije tanto en la URL, si no, en lo identifico del site donde se suplanta para robar las credenciales(aqui me estoy limitando a un tipo de Phishing eh)



Una de las cosas que mas complica (que no es tan dificil) es los templates en Gophish, aqui podemos encontrar una buena referencia https://docs.getgophish.com/user-guide/template-reference



Por otro lado, que tengamos que registrar un dominio o tener un espacio propio (server), no quiere decir que la campana debe salir cara, todo lo contrario, puede ser relativamente barata. + Info : [Offensive Security Cheatsheet](https://cheatsheet.haax.fr/phishing-redteam-and-se/phishing_campaign_setup/)



Por el contrario, si a tu organizacion le sobra el dinero, y quieres asistencia del todopoderoso `Kevin Mitnick` (esto es una broma folks) , podemos adquirir un software robusto llamado `KnowBe4` (https://www.knowbe4.com/) , esta empresa es dedicada al Social Engineering, se basa en campanas `sostificadas` de Phishing.



No te convence la pagina de KnowBe4 y todo lo que promete? aqui te dejo una foto de Mitnick, el gran hacker, con esto ha subido mi hacking level a Omni en HackTheBox.



<img src="https://pbs.twimg.com/profile_images/746860382225076224/qMcxl_W7_400x400.jpg" title="" alt="Follow Kevin Mitnick's (@kevinmitnick) latest Tweets / Twitter" data-align="center">





## <center>Consideraciones sobre Removable Media y otros menesteres.</center>



Entre las mitigaciones para los dispositivos externos, podemos aplicar politicas en nuestro DC, pero tambien existen soluciones como Network Access Control (NAC apartir de ahora)



El NAC viene al rescate de algo logico y de sentido comun, la seguridad fisica no es del todo adecuada para redes empresariales, aqui entra en funcion el NAC, es una solucion para implementar seguridad en la capa de Red (Modelo OSI por si no nos entendemos eh).



Con NAC, solo dispositivos autorizado pueden usar la red corporativa (esto no suena como ZeroTrust?...es un chiste...es un chiste). El administrador AKAK Lord of Network o Network Guy, necesita autorizar dispositivos.



Importante rescatar que para implementar NAC, el equipo necesita soportar NAC, podemos hablar de interoperabilidad pero aqui recaen temas como el IEEE 802.1X, pero no es el objetivo de esta entrada, inviten a su Network Guy favorito a un cafe y disfruten la charla.



Volviendo al punto anterior, uno de los problemas de NAC es que no todo lo soportan entre los que destacan : 

1. Network connected printers

2. IP Camera

3. VOIP phone



Lo que podemos hacer es poner estos ultimos (por Dios, es lo que el sentido comun nos dicta), en una VLAN separada y que no requiera authentication(siempre existe la authentication por MAC) , esta de mas decir que la VLAN debe estar segregada de la red corporativa.



**Nota**: Authentication por MAC no es lo mejor, solo basta tener conocimiento de Spoofing MAC 101.



<img src="https://mrsyiswhy.files.wordpress.com/2015/04/jpwr5.jpg?w=640" title="" alt="BYOD: Pervasive Computing Has Arrived | Postmodern Security" data-align="center">

Sin duda alguna **NAC** es una excelente opcion para esas empresas que tienen Bring Your Own Device (BYOD, que casi suena como aquella cancion de System Of a Down, que la B no es una D.... ), en este caso seria mas un MDM (Mobile Device Management) , para controlar aquellos dispositivos no reconocidos y darle acceso al ambiente.



El **MDM** es anti usuarios tontos, se puede forzar la authentication del dispositivo (asi como la complejidad del password), la presencia de aplicaciones que un usuario no debe de tener o hasta si hay un rooteo del dispositivo(no queremos este tipo de dispotivio en nuestra red).



### <center>Bypass al NAC: Ganandose la confianza del NAC o como ser el amigo de un amigo te puede ayudar.</center>



Anteriormente habia mencionado que no todo los dispositivos soportan NAC, y estos necesitan excepciones para operar en la red. Los actores maliciosos pueden explotar estas excepciones para obtener conectividad.



Por otra parte hay algunas herramientas creadas por ~~magos~~ RedTeamers con buenas ~~malas~~ intenciones como son PwnPlug o Fenrir, que abusan de admitir el modo de transparencia en el que abusan de la conectividad existente disponible para dispositivos autenticados/conocidos que han realizado primero la autenticación 802.IX [PWNplug - onsite Pen(-)tests, Reverse Shells, and Network Access Control - blog - because-security](https://blog.because-security.com/t/pwnplug-onsite-pen-tests-reverse-shells-and-network-access-control/235)



<img title="" src="data:image/png;base64,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" alt="802.1x NAC & Bypass Techniques" data-align="center" width="650">

Digamos que el escenario ideal seria :

Fenrir : 192.168.1.32

Workstation: 192.168.1.25



La ejecucion procederia en el siguiente orden : 

1. Un dispositivo autorizado realiza una autenticacion 802.1X

2. El dispositivo autorizado envia trafico HTTP a 192.168.20.30 (digamos que un web server interno) , Fenrir agrega una entrada (192.168.1.25:7698 ---> 192.168.20.30:80)

3. Fenrir se spoofea la MAC e IP del dispositivo autorizado y envia trafico HTTP hacia 192.168.20.31 (otro internal web server) , Fenrir agrega la entrada (192.168.1.26:8964 ----> 192.168.20.31:443)



El truco aqui recae en los record guardados, esto le permite a FENRIR entender que trafico esta retornando y reenviarlo al host autorizado.





## <center>Resumen NAC y MDM(un poco de 802.IX)</center>

<img title="" src="https://cdn.dopl3r.com//media/memes_files/que-se-repita-chica-diciendo-felizmente-wHrVK.jpg" alt="Que se repita, chica diciendo felizmente plantilla-" width="324" data-align="center">



Honestamente espero que no hayan leido todo lo anterior en cuanto a networking hablamos pero si lo hicieron, vamos a resumir un poco

IEEE 802.IX es un estándar de control de acceso a la red basado en puertos que puede ayudarnos a proteger nuestra red de dispositivos no deseados; diferentes proveedores ofrecen diferentes soluciones, así que utilice la que se adapte a su entorno. 

Las nuevas tecnologías incluyen MACsec y 802.IX-2010, que brindan mayor seguridad al implementar el cifrado de Capa 2 (Layer 2, que en gringo language suena mejor). https://standards.ieee.org/ieee/802.1X/4384/



NAC/MDM se usan para mejorar la postura de seguridad en los dispositivos corporativos (en orden de reforzar las politicas),  sin embargo, no todo los dispositivos soportan NAC o MDM, asi que algunas excepciones puedan que existan y necesitan ser manejadas, pues las mismas pueden ser byppaseadas.
