---
title :  "Connectar IoT Central con Digital Twins"
date : 2022-02-11T12:09:05+01:00
tags : ["IoT", "IoT Central", "Digital Twins"]
featured_image : "https://live.staticflickr.com/149/395471715_256795d6bd_4k.jpg"
description : ""
draft : true
---

[Azure IoT Central][iot-central] es una plataforma que ya preparada para conectar, visualizar y gestionar dispositivos de IoT en la nube, abstrayendo la complejidad tecnológica de una solución de IoT, en cuanto a conectividad, cuadros de mando, alertas, gestión de dispositivos, escalabilidad de la solución y un largo etcétera. Aunque la plataforma ya permite hacer muchísimas cosas, también es extensible y permite exportar los datos a otros elementos mediante reglas, [exportación de datos][iot-central-export] y una API REST.

[Azure Digital Twins][digital-twins] es un servicio PaaS para crear modelos digitales de un entorno y representar sus propiedades y relaciones. Los elementos de ese entorno pueden ser dispositivos contectados a nuestra red de IoT o cualquier otra cosa que necesitemos modelar. La plataforma permite actualizar, consutlar y visualizar los datos de los modelos en tiempo real.

En este artículo veremos cómo interconectar estos dos sistemas para ampliar las capacidades de IoT Central.

<!--more-->

## Introducción

En la documentación de [Azure Digital Twins][digital-twins-IoT] encontraremos un artículo sobre cómo conectar IoT Hub para la ingesta de la telemetría. El servicio de [IoT Central][iot-central] despliega por nosotros una serie de elementos como el IoT Provisioning Service, IoT Hub, Cosmos DB, una aplicación web y algunas cosas más, pero no tenemos acceso directo a esos elementos. Lo que podemos hacer es exportar la información mediante una serie de reglas.

Como ambas plataformas manejan información en tiempo real, vamos a utilizar la exportación de [Event Hubs][iot-central-eventhubs] para la interconexión de los dos sistemas.

## Creación de una app en IoT Central

Desde el portal de [creación de aplicaciones de IoT Central][iot-central-apps] vamos a generar una solución basada en una de las plantillas. Para este caso vamos a utilizar la solución Connected Logistics que ya trae simuladores de camiones paseándose por USA.

![Connected Logistics][iot-central-connected-logistics]

Durante la creación tendremos que proporcionar un nombre para la aplicación y el plan de precios. Yo he usado el gratuito que me permite probar la aplicación durante 7 días.

Al cabo de un rato los dos dispositivos simulados empezarán a generar datos y podremos ver esa información en el cuadro de mandos:
![Cuadro de mandos generado por IoT central con unos mapas que indican dónde están los camiones y gráficas con datos sbore los mismos][iot-central-northwind-traders-dashboard]


## Referencias

* Foto de portada: [Nodes, de Erwin Morales][nodes]
* [Digital Twins][digital-twins]
* [Ingestión de datos desde IoT Hub][digital-twins-iot]
* [Creación de apps en IoT Central][iot-central-apps]
* [Exportación de datos de IoT Central][iot-central-export]

[iot-central-connected-logistics]: iot-central-connected-logistics.png "Ejemplo de creación de aplicación Connected Logistics"
[iot-central-export]: iot-central-export.png "Exportación de datos en IoT Central"
[iot-central-export-destination]: iot-central-export-destination.png "Tipos de destino en la exportación de datos"
[iot-central-northwind-traders-dashboard]: iot-central-northwind-traders.png "La plantilla Connected Logistics genera una aplicación completa con un cuadro de mandos y dispositivos simulados"



[nodes]: https://www.flickr.com/photos/vonkinder/395471715/in/photolist-AWTXR-5Lpi3F-5Lpihe-5LpimZ-JHHw2H-5LpibD-5LphVx-5LphZ6-5Lpi6x-5LphSD-8vQPz6-9Wmyy6-3bfY2j-5KMs4q-2kiAWtf-ieiThA-b6VsA-idJuuN-4mj5yK-o4Lppi-9r2cT4-5CUbuo-8vQPme-5CUbyw-5fAtT1-9nHwPH-K7iqj7-D97x5L-amq1Ee-oBbUQo-7Vx1me-95scpo-sFf1KG-6WG3Nd-2dRJCmb-dYhGuZ-94QNrU-ePX9j-4gwLuA-PWoezQ-pFVBK3-p2wstK-p2wst4-pY69xc-pFQj6V-pY69sH-pFQj6p-p2twAE-p2wszX-pWauTw

[digital-twins]: https://azure.microsoft.com/en-us/services/digital-twins/#features "Gemelos Digitales"
[digital-twins-iot]: https://docs.microsoft.com/azure/digital-twins/how-to-ingest-iot-hub-data "Ingestion de datos de IoT Hub"
[iot-central]: https://docs.microsoft.com/azure/iot-central/core/overview-iot-central "Azure IoT Central"
[iot-central-apps]: https://apps.azureiotcentral.com/build "Apps de Azure IoT Central"
[iot-central-eventhubs]: https://docs.microsoft.com/en-us/azure/iot-central/core/howto-export-data?tabs=event-hubs%2Cjavascript%2Cconnection-string#set-up-an-export-destination "Exportación a Event Hubs"
[iot-central-export]: https://docs.microsoft.com/azure/iot-central/core/howto-export-data "Exportación de datos"