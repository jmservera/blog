+++
title =  "Reconectar la conexión WiFi automáticamente en Raspbian Buster"
date = 2020-11-01T22:04:27+01:00
tags = ["IoT","Raspberry Pi","Raspbian","Tips & Tricks"]
featured_image = "https://live.staticflickr.com/3706/19346279608_9805ba6259_b.jpg"
description = ""
draft = false
+++

En casa hemos ido conectando el jardín, las luces, persianas, puertas, la tele, el aire acondicionado e incluso a Higgs, nuestro conejo, aunque esto ya os lo explicaré otro día. Tenemos una gran variedad de dispositivos y sensores, entre ellos unas cuantas Raspberry Pi 2 que se conectan a la red WiFi. Ya hemos tenido que reiniciar muchas veces las Raspis headless después de un corte de conexión, porque parece ser que el servicio dhcpcd no gestiona bien esta situación.

Tras realizar una búsqueda he encontrado diversos métodos más o menos complicados, el que más me gustó por la sencillez fue el de [Alex Bain](http://alexba.in/blog/2015/01/14/automatically-reconnecting-wifi-on-a-raspberrypi/), así que lo he adaptado un poco para que funcione bien en Raspbian Buster, que es la distribución que utilizo.

Lo que hay que hacer es crear un script en */usr/local/bin/wifi_rebooter.sh*. Sería algo como el código que hay a continuación, cambiando la dirección a la que queréis hacer ping para comprobar la conexión y el nombre del interfaz, el primero suele ser *wlan0*:

```bash
#!/bin/bash

# The IP for the server you wish to ping (192.168.1.1 is typically your router)
SERVER=192.168.1.1

# Only send two pings, sending output to /dev/null
ping -c2 ${SERVER} > /dev/null

# If the return code from ping ($?) is not 0 (meaning there was an error)
if [ $? != 0 ]
then
    printf '#' >> /tmp/wirelesscount
    ln=$(stat --printf="%s" /tmp/wirelesscount)
    if [ $ln -gt 2 ]
    then
        # third time reboot raspi if everything else fails
        cat /dev/null > /tmp/wirelesscount
        reboot now
    fi
    # Restart the wireless interface
    ip link set wlan0 down
    ip link set wlan0 up
else
    # reset count if it worked
    cat /dev/null > /tmp/wirelesscount
fi
```

Después le damos permisos de ejecución:

```bash
sudo chmod +x /usr/local/bin/wifi_rebooter.sh
```

Y finalmente añadimos un intervalo de 5 minutos en */etc/crontab* para comprobar la conexión, si pierde el ping reiniciará el interfaz de red:

```bash
*/5 *   * * *   root    /usr/local/bin/wifi_rebooter.sh
```

Así, por fin ya no tenemos que ir a reiniciar manualmente todas las Raspis cuando han tenido algún corte de conexión.

Makers gonna make.
