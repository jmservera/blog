---
title :  "Connectar IoT Central con Digital Twins"
date : 2022-02-11T12:09:05+01:00
tags : ["IoT", "IoT Central", "Digital Twins"]
featured_image : "https://live.staticflickr.com/149/395471715_256795d6bd_4k.jpg"
description : ""
draft : true
---

Con [Azure IoT Central][iot-central] podemos poner en marcha una plataforma de IoT con cuadros de mando, gestión de dispositivos, alertas, comunicación bidireccional y muchas cosas más en cuestión de minutos. Además la plataforma es extensible y permite exportar los datos a otros elementos mediante reglas, [exportación de datos][iot-central-export] y una API REST.

[Azure Digital Twins][digital-twins] es un servicio PaaS para crear modelos digitales de un entorno y representar sus propiedades y relaciones. Los elementos de ese entorno pueden ser dispositivos contectados a nuestra red de IoT o cualquier otra cosa que necesitemos modelar. La plataforma permite actualizar, consultar y visualizar los datos de los modelos en tiempo real.

En este artículo veremos cómo interconectar estos dos sistemas para ampliar todavía más las capacidades que ya tiene IoT Central.

<!--more-->

## Introducción

En la documentación de [Azure Digital Twins][digital-twins-IoT] encontraremos un artículo sobre cómo conectar IoT Hub para la ingesta de la telemetría. El servicio de [IoT Central][iot-central] despliega por nosotros una serie de elementos como el IoT Provisioning Service, IoT Hub, Cosmos DB, una aplicación web y algunas cosas más, pero no tenemos acceso directo a esos elementos. Lo que podemos hacer es exportar la información mediante una serie de reglas.

Como ambas plataformas manejan información en tiempo real, vamos a utilizar la exportación de [Event Hubs][iot-central-eventhubs] para la interconexión de los dos sistemas.

Antes de ponernos manos a la obra vamos a ver algo de la terminología que vamos a utilizar:

* [Digital Twin Definition Language][digital-twins-DTDL] o DTDL, es un lenguaje para modelar dispositivos de IoT "plug and play" y también gemelos digitales. Es decir, vamos a usar el mismo lenguaje de definición en ambos sistemas.
* [Propiedades][digital-twins-properties] son las características de un elemento digital.

## Pasos previos

Como os he comentado, necesitamos un Event Hubs donde volcar los datos en tiempo real de IoT Hub, así que primero necesitamos tener creado ese elemento:

[todo: crear Event Hubs]

Y ya que estamos creando recursos, vamos también a crear un servicio de Digital Twins

Para conectar al Event Hubs necesitaremos una cadena de conexión que podemos obtener en ... Y tenemos que crear un grupo de lectura, de forma que no pisemos a otras aplicaciones que se conecten al Event Hubs.

[todo: event hubs connection string screenshot]

Nos guardaremos esas credenciales para más adelante.

[todo: crear Digital Twins]

## Creación de una app en IoT Central

Desde el portal de [creación de aplicaciones de IoT Central][iot-central-apps] vamos a generar una solución basada en una de las plantillas. Para este caso vamos a utilizar la solución Connected Logistics que ya trae simuladores de camiones paseándose por USA.

![Connected Logistics][iot-central-connected-logistics]

Durante la creación tendremos que proporcionar un nombre para la aplicación y el plan de precios. Yo he usado el gratuito que me permite probar la aplicación durante 7 días.

Al cabo de un rato los dos dispositivos simulados empezarán a generar datos y podremos ver esa información en el cuadro de mandos:
![Cuadro de mandos generado por IoT central con unos mapas que indican dónde están los camiones y gráficas con datos sbore los mismos][iot-central-northwind-traders-dashboard]

Ahora vamos a conectar IoT Central para con el resto de servicios para poder tener la información en Digital Twins.

## Exportación

En IoT Central no tenemos un acceso directo a IoT Hub, así que la opción que tenemos para poder obtener los datos en tiempo real es crear una regla de exportación a un EventHubs.

![Destino][iot-central-export-destination]

Usaremos la cadena de conexión que hemos generado antes en 

![Event Hubs Config][iot-central-export-eh-destination]

![Mensaje recibido sin nombre][event-hub-message-received]

![Añadir columnas][iot-central-export-enrichments]

## Referencias

* Foto de portada: [Nodes, de Erwin Morales][nodes]
* [Digital Twins][digital-twins]
* [Ingestión de datos desde IoT Hub][digital-twins-iot]
* [Creación de apps en IoT Central][iot-central-apps]
* [Exportación de datos de IoT Central][iot-central-export]
* [Referencia del DTDL][digital-twins-DTDL]
* [Plantillas de dispositivo en IoT Central][iot-central-define-device]

[event-hub-message-received]: event-hub-message-received.png "Event Hubs Message Received"
[iot-central-connected-logistics]: iot-central-connected-logistics.png "Ejemplo de creación de aplicación Connected Logistics"
[iot-central-export]: iot-central-export.png "Exportación de datos en IoT Central"
[iot-central-export-destination]: iot-central-export-destination.png "Exportación de datos en IoT Central"
[iot-central-export-eh-destination]: iot-central-export-eh-destination.png "Configuración del destino de datos"
[iot-central-export-enrichments]: iot-central-export-enrichments.png "Enriquecimiento de la exportación de datos"

[iot-central-export-destination]: iot-central-export-destination.png "Tipos de destino en la exportación de datos"
[iot-central-northwind-traders-dashboard]: iot-central-northwind-traders.png "La plantilla Connected Logistics genera una aplicación completa con un cuadro de mandos y dispositivos simulados"



[nodes]: https://www.flickr.com/photos/vonkinder/395471715/in/photolist-AWTXR-5Lpi3F-5Lpihe-5LpimZ-JHHw2H-5LpibD-5LphVx-5LphZ6-5Lpi6x-5LphSD-8vQPz6-9Wmyy6-3bfY2j-5KMs4q-2kiAWtf-ieiThA-b6VsA-idJuuN-4mj5yK-o4Lppi-9r2cT4-5CUbuo-8vQPme-5CUbyw-5fAtT1-9nHwPH-K7iqj7-D97x5L-amq1Ee-oBbUQo-7Vx1me-95scpo-sFf1KG-6WG3Nd-2dRJCmb-dYhGuZ-94QNrU-ePX9j-4gwLuA-PWoezQ-pFVBK3-p2wstK-p2wst4-pY69xc-pFQj6V-pY69sH-pFQj6p-p2twAE-p2wszX-pWauTw

[digital-twins]: https://azure.microsoft.com/en-us/services/digital-twins/#features "Gemelos Digitales"
[digital-twins-DTDL]: https://github.com/Azure/opendigitaltwins-dtdl/blob/master/DTDL/v2/dtdlv2.md "Digital Twin Definition Language"
[digital-twins-iot]: https://docs.microsoft.com/azure/digital-twins/how-to-ingest-iot-hub-data "Ingestion de datos de IoT Hub"
[iot-central]: https://docs.microsoft.com/azure/iot-central/core/overview-iot-central "Azure IoT Central"
[iot-central-apps]: https://apps.azureiotcentral.com/build "Apps de Azure IoT Central"
[iot-central-define-device]: https://docs.microsoft.com/en-us/azure/iot-central/core/howto-set-up-template#create-a-device-template-from-the-device-catalog "Creación de una plantilla de dispositivo"
[iot-central-eventhubs]: https://docs.microsoft.com/en-us/azure/iot-central/core/howto-export-data?tabs=event-hubs%2Cjavascript%2Cconnection-string#set-up-an-export-destination "Exportación a Event Hubs"
[iot-central-export]: https://docs.microsoft.com/azure/iot-central/core/howto-export-data "Exportación de datos"