---
title: "005 Instalar Unifi Nvr Antiguo"
date: 2021-02-28T11:53:16+01:00
draft: true
tags: ["IoT"]
---

Hace poco Unifi descatalogó su Network Video Recorder (NVR) en favor del nuevo sistema Unifi Protect, y, aunque tiene muy buena pinta, ya no proporcionan un instalador sino que está completamente ligado a su hardware. Como esto no cuadra bien con toda la instalación que tenemos sobre Home Assistant, vamos a seguir usando el NVR conectado por RTSP al hassio, pero con la última actualización de seguridad de Java 8 también ha dejado de funcionar. Así que hoy aprenderemos a instalar un NVR y fijar la instalación de Java a una versión en concreto en un Ubuntu 20.10.

<!--more-->

> **Aviso de seguridad**: Como Unifi no ha abierto el software NVR, este procedimiento nos obliga a utilizar una versión obsoleta y no mantenida de NVR y de Java. Con todas las consecuencias de seguridad que puede tener es altamente recomendable proteger al máximo esa instalación, dentro de una VM o contenedor, sin dejar ningún puerto abierto al exterior, únicamente lo necesario para que el HASSIO se pueda comunicar con el sistema.

## Paso 1, fijar la versión de Java en Ubuntu

NVR sólo funciona bien con la versión de Java 8u275, así que el comando habitual de instalación de Java 8 no nos servirá, porque la versión actual es la 8u282, así que tendremos que hacer dos cosas: instalar la versión que nos interesa y luego impedir la actualización, porque cualquier operación de `bash apt upgrade` nos actualizaría esa versión y dejará de funcionar el NVR. Lo malo es que el error del NVR será bastante críptico, pues no nos dice que no funciona Java sino que nos dará un simple *...fail!*:

```log
Feb 26 12:43:48 nvr systemd[1]: Starting LSB: Ubiquiti unifi-video...
Feb 26 12:43:49 nvr unifi-video[535]:  * Starting Ubiquiti UniFi Video unifi-video
Feb 26 12:43:50 nvr unifi-video[630]: (unifi-video) Hardware type:Unknown
Feb 26 12:43:50 nvr unifi-video[630]: (unifi-video) checking for system.properties and truststore files...
Feb 26 12:43:51 nvr unifi-video[535]:    ...fail!
Feb 26 12:43:51 nvr systemd[1]: Started LSB: Ubiquiti unifi-video.
```

Lo bueno es que tiene solución, así que lo que vamos a hacer es:

1. Descargar el paquete Java 8u275:

https://packages.debian.org/stretch/amd64/openjdk-8-jre-headless/download

```bash
wget https://packages.debian.org/stretch/amd64/openjdk-8-jre-headless/download
sudo apt install 
openjdk-8-jre-headless_8u275-b01-1~deb9u1_amd64.deb

```

2. Fijar esa versión para que un apt upgrade no nos actualice a una versión más nueva:

```bash
sudo apt-mark hold openjdk-8-jre-headless 
```

## Instalar NVR

Aunque no esté mantenida, Unifi sigue teniendo disponible como descarga las últimas versiones del software. Como hay que comprobar las versiones de Mongo, Java, etc., [Glenn .R creó unos scripts de instalación](https://glennr.nl/scripts).

Durante la instalación nos preguntará por si queremos actualizar automáticamente el software de Unifi, a lo que tendremos que contestar que no, pues añade unas fuentes que ya no existen y nos rompería las actualizaciones del resto de paquetes de Ubuntu.


