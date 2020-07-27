+++
title =  "Cómo montar un servicio de video chat"
date = 2020-06-24T10:25:22+02:00
tags = ["Azure","WebRTC","TURN","PoC","ARM Templates"]
featured_image = "https://live.staticflickr.com/207/515106921_a83de99560_o.jpg"
description = "Si Doug pudo en 1968 tú también puedes"
draft = true
+++


Aunque nos parezca algo muy moderno, en los años 60 ya se empezaban a comercializar los primeros sistemas de videoconferencia, y ya había gente como Douglas Engelbart que te hacía [explotar la cabeza en el 68 con sus demos][motherofalldemos].
![Douglas Engelbart haciendo LA DEMO](/como-montar-videochat/engelbartdemo.png)
Ya avanzado el siglo XXI ¿Cómo de difícil sería montarnos un videochat privado usando WebRTC serverless tal como anunciaba Western Electric? Como suele ocurrir, ni es tan sencillo, ni es verdad que se pueda hacer una conexión P2P fácilmente entre dos puntos.

## WebRTC

Hoy en día, tenemos en nuestro navegador la tecnología que nos permite comunicarnos con otro navegador de forma directa, es decir, podemos enviar datos entre dos clientes sin necesidad de un servidor, siempre que se cumplan ciertas condiciones. Por ejemplo, podríamos construir una solución muy sencilla tal como explican en el artículo [WebRTC and Node.js][1] y ya dispondríamos de un chat de vídeo completamente funcional. Si seguimos la guía y [actualizamos algunas librerías][videochatgit], funcionará al probarlo con dos navegadores que estén en nuestra red local, pero si intentamos usarlo entre dos usuarios en redes distintas probablemente no conseguiremos conectarnos.

[![India, Delhi - Indian-style cable spaghetti - February 2018 by Cyprien Hauser](https://live.staticflickr.com/65535/49204696853_6df9abbc5c_c.jpg)](https://flic.kr/p/2hY3W7i)

Esto ocurre porque entre los dos puntos seguramente haya Firewalls, NAT, Proxies y muchas otras cosas que impiden una conexión directa. Ya existen suficientes artículos que explican cómo funciona, así que no me voy a extender demasiado en los detalles y os dejo unos enlaces a las diferentes partes que solucionan los problemas que vamos a encontrarnos:

* [STUN, o Session Traversal Utilities for NAT][STUN], es el conjunto de herramientas que necesitaremos para que los dos clientes remotos puedan encontrarse a través de la maraña de redes del Mundo Real&trade.
* [TURN Server, o Traversal Using Relays around NAT][TURN], cuando STUN no es suficiente necesitaremos que un servidor haga de intermediario.
* [ICE, Interactive Connectivity Establishment][ICE], es la forma que tiene WebRTC de decidir si puede conectar directamente, si no intentará usar STUN para saltar el NAT y finalmente usara TURN si no hay alternativa (TURN es la alternativa más cara porque hay que transferir el vídeo a través del servidor intermedio).

## Show me the code

Si habéis leído los artículos, al parecer sí que necesitamos un servidor de streaming. Aunque en principio es sólo "por si acaso" los dos navegadores no pueden establecer una conexión directa, la realidad es que la mayor parte de las veces nos va a ocurrir eso. La buena noticia es que existe un proyecto open source que despliega un servidor TURN / STUN / ICE llamado [coturn][coturngit] y ni siquiera lo tenemos que compilar porque ya está disponible como [paquete para Ubuntu][coturn].

Mi primera reacción fue intentar montar todo eso en un contenedor y desplegarlo como una Azure Container Instance. El problema es que ACI permite abrir múltiples puertos, pero no en múltiples protocolos, tienes que elegir si quieres UDP o TCP, pero no pueden ser ambos, así que por ahora vamos a tener que usar una máquina virtual para hacer la prueba.

## Créditos

* Foto de portada por Daniel Yanes Arroyo [Western Electric is crossing a telephone with a TVset][pinkponk].


[motherofalldemos]: https://www.youtube.com/watch?v=M5PgQS3ZBWA&list=PLCGFadV4FqU3flMPLg36d8RFQW65bWsnP
[1]: https://tsh.io/blog/how-to-write-video-chat-app-using-webrtc-and-nodejs/
[STUN]: https://en.wikipedia.org/wiki/STUN
[TURN]: https://en.wikipedia.org/wiki/Traversal_Using_Relays_around_NAT
[ICE]: https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment
[coturngit]: https://github.com/coturn/coturn
[coturn]: https://packages.ubuntu.com/bionic/coturn "paquete para Ubuntu"
[videochatgit]: https://github.com/jmservera/videochat
[Azure]: https://azure.microsoft.com
[pinkponk]: https://www.flickr.com/photos/pinkponk/515106921