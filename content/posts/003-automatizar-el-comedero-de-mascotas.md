+++
title =  "Automatiza el comedero de tu mascota para poder irte de vacaciones tranquilo"
date = 2020-07-31T10:25:22+02:00
tags = ["IoT","Maker","3D Printing","Tasmota"]
featured_image = "/automatizar-comedero/cookieboard_bb.png"
description = ""
draft = true
+++

Ahora que parece que algún día podremos empezar a viajar otra vez, tendremos que encontrar soluciones para cuando no estemos en casa y que los seres vivos que no nos podemos llevar con nosotros sobrevivan durante nuestra ausencia.

## Circuito para control del motor

## Tornillo de Arquímedes

## Materiales

Necesitamos:

### Para la electrónica

* Una placa de [prototipado PCB][electrocookie] 
* Un [ESP-01S][ESP01S], preferiblemente con adaptador para breadboard, pero nosotros lo soldamos directamente.
* Usamos [cable estañado de calibre 22][tuofeng22] para hacer las conexiones
* Módulo conversor de 12v a 3v [MP1584EN][mp1584en]
* El driver de los motores es un [A4988 o Pololu A1][pololua1], aunque dependerá de qué motor utilicéis
* Dos condensadores de 1&micro;F, aunque vale la pena [comprar un surtido][capacitors]
* Una fuente de alimentación de 12VDC de unos 4 o 5 amperios, pensad que alimentaremos el motor y el ESP01, nosotros la montamos directamente en el cuadro eléctrico con una [fuente para rail DIN][dinrail12v]
* Adaptador USB para programar el ESP01, me gusta especialmente [este que lleva un interruptor][esp01usb] para puentear el GPIO0 con GND y poner el ESP en modo programación.
* [Conector de corriente][powerplug]

![Circuito][cookieboard]

### Para la parte mecánica

* Un motor paso a paso que tenga buen torque, yo he usado los [Nema 17 de 2A][nema17]
* [Tornillos M3][m3screw] para agarrar el motor a la caja.
* [Tuercas moleteadas][m3nuts] de tamaño M3 que insertaremos en la caja y aguantarán el motor
* El resto normalmente podremos imprimirlo en 3D y usar algunos tubos de PVC. Aquí dependerá de qué proyecto os guste más, nosotros hemos usado [el feeder de NorthenLayers][northenlayers], pero podéis encontrar muchos más como el [Gyver's cat feeder II screw][gyverscrew]

[capacitors]: https://amzn.to/3jnIYcj
[dinrail12v]: https://amzn.to/32GCoad
[electrocookie]: https://amzn.to/2QBRGr0
[ESP01S]: https://amzn.to/32GzQJb
[esp01usb]: https://amzn.to/3jsgi1L
[m3nuts]: https://amzn.to/2GdGbEp
[m3screw]: https://amzn.to/2YKnHBN
[mp1584en]: https://amzn.to/34QHcfT
[nema17]: https://amzn.to/2EtQVOp
[tuofeng22]: https://amzn.to/2GdHuTP
[pololua1]: https://amzn.to/31Hqw8C
[powerplug]: https://amzn.to/3bblhkk

[cookieboard]: /automatizar-comedero/cookieboard_bb.png "Cookie board"
[gyverscrew]: https://www.thingiverse.com/thing:4753380
[northenlayers]: https://www.thingiverse.com/thing:958180 "Auger-based feeder for animals, fish etc using Nema 17 stepper motor"