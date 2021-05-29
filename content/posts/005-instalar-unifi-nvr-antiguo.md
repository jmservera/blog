---
title: "Cómo usar las cámaras Unifi en Home Assistant tras la obsolescencia de su NVR"
date: 2021-05-30T00:21:08+02:00
featured_image: https://live.staticflickr.com/3132/5843649056_399705f3d0_w.jpg
draft: false
tags: ["IoT"]
---

Unifi descatalogó su [Network Video Recorder (NVR)][nvr] en favor del nuevo sistema [Unifi Protect][unifi-protect], y para éste último ya no proporcionan un instalador que podamos instalar en nuestro propio sistema, como hacían antes con Unifi Video. Ahora sólo lo proporcionan junto con su hardware. Como nosotros acabábamos de renovar el hardware donde instalamos el [Home Assistant][hassio] y unas cuantas cosas más para nuestra red local, no nos apetece comprar un hardware que en realidad no necesitamos, así que hoy aprenderemos a instalar un NVR y fijar la instalación de Java a la versión donde funciona ese software en un Ubuntu 20.10.

<!--more-->

> **Aviso de seguridad**: Como Unifi no ha abierto el software NVR, este procedimiento nos obliga a utilizar una versión obsoleta y no mantenida tanto de NVR como de Java. Con todas las consecuencias de seguridad que puede tener es altamente recomendable proteger al máximo esa instalación, dentro de una VM o contenedor, sin dejar ningún puerto abierto al exterior, únicamente lo necesario para que el HASSIO se pueda comunicar con el sistema.

También han descatalogado la plataforma cloud que permitía ver las cámaras desde su propia aplicación, sin necesidad de tener una IP fija ni abrir puertos en nuestro sistema y hay que usar la nueva que sólo funciona con Unifi Protect. Por suerte, nosotros usamos Home Assistant y ya teníamos eso resuelto con su fantástica [aplicación de móvil][app-hassio], que nos permite crear un interfaz personalizado y visualizar las cámaras en directo. No tiene toda la funcionalidad, por ejemplo, no podemos controlar el zoom de las cámaras desde el Home Assistant, pero de eso ya hablaremos otro día.

Para seguir teniendo acceso a las cámaras tenemos varias opciones. La más sencilla es eliminar el NVR, poner las cámaras en modo standalone y conectar directamente las cámaras al Home Assitant a través de RTSP. Con esta opción perdemos la detección de movimiento y la capacidad de grabar, si eso no te interesa puedes ignorar esta guía y conectar directamente las cámaras con una configuración parecida a esta:

```yaml
camera:
    platform: generic
    name: mycam
    stream_source: rtsp://yourcamerauri
    verify_ssl: false
```

La segunda opción es mantener el NVR, para ello tenemos que realizar los tres pasos que veremos a continuación para poder mantener ese software funcionando. Otra opción es usar otro NVR como [Shinobi][Shinobi], del que también hablaremos otro día.



## Paso 1, fijar la versión de Java en Ubuntu

NVR se puede instalar en las últimas versiones de Ubuntu, pero sólo funciona bien con la versión de **Java 8u275**, así que el comando habitual de instalación de Java 8 no nos servirá, porque la versión actual a día de hoy es la 8u282, así que tendremos que hacer dos cosas: 

1. Forzar la instalación de la versión que nos interesa
2. Impedir la actualización automática de la misma cuando ejecutemos cualquier operación de `bash apt upgrade`, para evitar que deje de funcionar el NVR.

> Lo malo del error del NVR es que no da mucha información, pues no nos indica que no funciona con esa versión de Java sino que nos dará un simple mensaje con el texto *...fail!*:

```log
Feb 26 12:43:48 nvr systemd[1]: Starting LSB: Ubiquiti unifi-video...
Feb 26 12:43:49 nvr unifi-video[535]:  * Starting Ubiquiti UniFi Video unifi-video
Feb 26 12:43:50 nvr unifi-video[630]: (unifi-video) Hardware type:Unknown
Feb 26 12:43:50 nvr unifi-video[630]: (unifi-video) checking for system.properties and truststore files...
Feb 26 12:43:51 nvr unifi-video[535]:    ...fail!
Feb 26 12:43:51 nvr systemd[1]: Started LSB: Ubiquiti unifi-video.
```

La buena noticia es que tiene solución, así que lo que vamos a hacer es:

1. Descargar el paquete [Java 8u275][java8u275]:

```bash
wget https://packages.debian.org/stretch/amd64/openjdk-8-jre-headless/download
sudo apt install 
openjdk-8-jre-headless_8u275-b01-1~deb9u1_amd64.deb
```

2. Marcar esa versión en nuestro sistema operativo para que un apt upgrade no nos actualice a una versión más nueva:

```bash
sudo apt-mark hold openjdk-8-jre-headless 
```

## Paso 2: Instalar NVR

Aunque no esté mantenida, Unifi sigue teniendo disponible como descarga las últimas versiones del software. Instalarlo nunca ha sido muy fácil porque hay que comprobar las versiones de Mongo, Java, y el resto del software necesario, porque si no está todo correcto, nos fallará con errores bastante ininteligibles. Por suerte, [Glenn .R creó unos scripts de instalación][glenn] que comprueban todo eso e instalan el NVR.

Durante la instalación nos preguntará  si queremos actualizar automáticamente el software de Unifi, a lo que tendremos que contestar que no, pues añade unas fuentes que ya no existen y nos rompería las actualizaciones del resto de paquetes de Ubuntu.

Una vez instalado, ya tenemos nuestro NVR en una IP interna desde la que podremos acceder al puerto 7080 y conectar con todas las cámaras.

## Paso 3: Conectar con Home Assitant

Para poder añadir el stream de las cámaras en una pestaña del Home Assistant necesitamos habilitar el streaming RTSP y obtener la clave de API y, opcionalmente, el password de gestión de las cámaras.

En el portal del NVR, en la configuración general podemos habilitar el protocolo RTSP:

![La opción Streaming Ports, tiene un interruptor para activar RTSP y un cuadro donde poner el puerto, que por defecto es 7447][RTSP]

También necesitaremos la clave de API, la podemos encontrar en la configuración de usuario:

![En /users podemos abrir la configuración del usuario, activar el acceso a API y copiar el contenido del campo API Key][api-key]

Además, podemos también copiar el password que el NVR les va a asignar a las cámaras una vez configuradas. Este password lo podemos encontrar en la configuración general en el apartado "camera settings". Necesitaremos pulsar sobre el campo para ver el password:

![En "camera settings" está el password que se usa en las cámaras, aunque está escondido, al pulsar sobre él se muestra][camera-password]

Y, para acabar, en el archivo de configuración del Home Assistant usaremos estos datos en la sección camera, porque ya existe una integración con el NVR de Unifi llamada *uvc*:

```yaml
camera:
  - platform: uvc
    nvr: [NVR IP]
    key: [API KEY]
    password: [CAMERA PASSWORD]
    ssl: [recomendable poner true]
```

> **Zero Trust**: Sí, aunque sea dentro de la red local de nuestra propia casa, siempre es conveniente utilizar HTTPS y proteger todos nuestros sistemas. Pensad en todos los cacharros *"SMART"* que tenéis conectados a vuestra red local: el móvil, la tele, la aspiradora o la tostadora esa que usa Java, cualquier cacharro puede estar espiando vuestra red local sin que os déis cuenta.

Una vez reiniciéis el Home Assistant, en las integraciones deberían aparecer las cámaras que tengáis configuradas en el NVR:

![Listado de cámaras en el Home Assistant][cameras]

## Otras cosas interesantes

Para Unifi Video hay algunos elementos adicionales que nos mejorarán mucho la experiencia en el Home Assistant:

* [Eventos MQTT][unifi-video-mqtt]: podemos generar eventos MQTT y crear en el Home Assistant sensores que realicen acciones, por ejemplo cuando se detecta un movimiento en una cámara.
* [Creación de GIFs][unifi-video-gifs]: podéis generar pequeñas imágenes en movimiento y recibirlas directamente en un bot de Telegram, por ejemplo.
* [Video to Telegram][video2telegram]: como convertir a GIF puede necesitar bastante proceso, podéis hacer la conversión y el envío en Azure.

## No future

Como NVR ha llegado al final de su ciclo de vida, esto es sólo un apaño temporal, estamos investigando otras alternativas que además nos permitan usar cualquier tipo de cámara, así que espero, dentro de poco, tener un artículo más extenso sobre crear nuestro propio recorder con herramientas Open Source como [Shinobi] o similares.

## Resumen de enlaces

* [App del Home Assistant][app-hassio]
* [Scripts de Glenn para instalar NVR v3.10.13][glenn]
* [Home Assistant][hassio]
* [Paquete de Java 8 275][java8u275]
* [Fin de vida del NVR][nvr]
* [Unifi protect][unifi-protect]
* [Shinobi][Shinobi]
* [Eventos MQTT de Unifi Video][unifi-video-mqtt]


## Credits

* Header picture by [wiredforlego][surveillance-cam].

[app-hassio]: http://hassioapp "La aplicación móvil oficial para Home Assistant"
[glenn]: https://get.glennr.nl/unifi-video/install/unifi-video-3.10.13.sh
[hassio]: https://home-assistant.org
[java8u275]: https://packages.debian.org/stretch/amd64/openjdk-8-jre-headless/download
[nvr]: https://help.ui.com/hc/en-us/articles/360057458834-Accessing-UniFi-Video-after-End-of-Support
[unifi-protect]: http://amazon.unifi.protect
[unifi-video-gifs]: https://github.com/fabtesta/unifi-nvr-api-motion-mqtt-gifs
[unifi-video-mqtt]: https://github.com/mzac/unifi-video-mqtt
[Shinobi]:https://shinobi.video/
[video2telegram]: https://github.com/jmservera/AzureBlob2Telegram

[cameras]: /005-nvr/cameras.png "Listado de cámaras en el Home Assistant"
[camera-password]: /005-nvr/camerapassword.png "En 'camera settings' está el password que se usa en las cámaras, aunque está escondido, al pulsar sobre él se muestra"
[RTSP]: /005-nvr/RTSP.png "La opción Streaming Ports, tiene un interruptor para activar RTSP y un cuadro donde poner el puerto, que por defecto es 7447"
[surveillance-cam]: https://www.flickr.com/photos/wiredforsound23/ "Surveillance Camera"
[api-key]: /005-nvr/apikey.png "En /users podemos abrir la configuración del usuario, activar el acceso a API y copiar el contenido del campo API Key"
