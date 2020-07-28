+++
title =  "Cómo montar un servicio de videochat"
date = 2020-06-24T10:25:22+02:00
tags = ["Azure","WebRTC","TURN","PoC","ARM Templates"]
featured_image = "https://live.staticflickr.com/207/515106921_a83de99560_o.jpg"
description = "Si Doug pudo en 1968 tú también puedes"
draft = false
+++


Aunque nos parezca algo muy moderno, en los años 60 ya se empezaban a comercializar los [primeros sistemas de videoconferencia][pinkponk], y ya había gente como Douglas Engelbart que te hacía [explotar la cabeza en el 68 con sus demos][motherofalldemos].
![Douglas Engelbart haciendo "La madre de todas las demos"](/como-montar-videochat/engelbartdemo.png "La demo de la lista de la compra de Doug")
Ya avanzado el siglo XXI ¿Cómo de difícil sería montarnos un videochat privado usando WebRTC serverless tal como anunciaba Western Electric? Como suele ocurrir, ni es tan sencillo, ni es verdad que se pueda hacer una conexión P2P fácilmente entre dos puntos.

## WebRTC

Hoy en día, tenemos en nuestro navegador la tecnología que nos permite comunicarnos con otro navegador de forma directa, es decir, podemos enviar datos entre dos clientes sin necesidad de un servidor, siempre que se cumplan las [condiciones ideales][spherical_cow]. Por ejemplo, podríamos construir una solución muy sencilla tal como explican en el artículo [WebRTC and Node.js][webrtcdemo] y ya dispondríamos de un chat de vídeo bastante funcional. Si seguimos la guía y [actualizamos algunas bibliotecas][videochatgit], funcionará al probarlo con dos navegadores que estén en nuestra red local, pero si intentamos usarlo entre dos usuarios en redes distintas probablemente no conseguiremos conectarnos.

[![India, Delhi - Indian-style cable spaghetti - February 2018 by Cyprien Hauser](https://live.staticflickr.com/65535/49204696853_6df9abbc5c_c.jpg)](https://flic.kr/p/2hY3W7i "A ver si encuentras qué cable es el tuyo")

Esto ocurre porque entre los dos puntos seguramente haya firewalls, NAT, algunos proxies y muchas otras cosas que impiden una conexión directa. Ya existen suficientes artículos que explican cómo funciona, así que no me voy a extender demasiado en los detalles y os dejo unos enlaces a los problemas y su soluciones:

* [STUN, o Session Traversal Utilities for NAT][STUN], es el conjunto de herramientas que necesitaremos para que los dos clientes remotos puedan encontrarse a través de la maraña de redes del Mundo Real&trade;.
* [TURN Server, o Traversal Using Relays around NAT][TURN], cuando STUN no es suficiente necesitaremos que un servidor haga de intermediario.
* [ICE, Interactive Connectivity Establishment][ICE], es la forma que tiene WebRTC de decidir si puede conectar directamente, si no intentará usar STUN para saltar el NAT y finalmente usara TURN si no hay alternativa (TURN es la alternativa más cara porque hay que transferir el vídeo a través del servidor intermedio).

## Show me the code

Tal como explican los artículos, al parecer sí que necesitamos un servidor de streaming. Aunque en principio es sólo "por si acaso" los dos navegadores no pueden establecer una conexión directa, la mayor parte de las veces tendremos que recurrir al servidor TURN. La buena noticia es que existe un proyecto open source que despliega un servidor TURN / STUN / ICE llamado [coturn][coturngit] y ni siquiera lo tenemos que compilar porque ya está disponible como [paquete para Ubuntu][coturn], la mala es que, para escenarios de producción, esto saldrá bastante caro, pues necesitarás una infraestructura potente que además permita escalar horizontalmente.

Desplegar esto en Azure es relativamente sencillo. La aplicación web la desplegaremos en [Azure App Service][appservice], que ya nos permitirá la alta disponibilidad, gestión de certificados e incluso podremos gestionar la seguridad de acceso usando [Azure Active Directory][aad] con muy pocos cambios en nuestra solución y que veremos en otro artículo más adelante. El servidor TURN podríamos desplegarlo en [Azure Container Instance][aci], pero para que funcione al 100% necesitamos que responda tanto en UDP como en TCP y eso todavía no está soportado en ACI, así que lo desplegaremos en una [Máquina Virtual][vm] con un script de inicio.

### Servidor TURN

Lo primero que vamos a montar es la VM del servicio TURN con un script de inicio que configurará la aplicación *coturn* y generará unos certificados para la misma. La primera parte del script instala las aplicaciones necesarias:

```bash
if [ ! $4 ]
then
    echo "Usage: $0 [adminusername] [adminpassword] [serverFQDN] [externalip]"
    echo "Ensure the external IP does not change over time"
fi

# first get all the files
sudo add-apt-repository ppa:certbot/certbot -y
sudo apt-get update && sudo apt-get install -y dnsutils coturn certbot

sudo systemctl stop coturn

# enable service
sudo echo "TURNSERVER_ENABLED=1" > /etc/default/coturn
```

Luego generamos el fichero de configuración en base a los parámetros que llegan al script (podéis ver los parámetros más arriba en el script anterior):

```bash
# create configuration
echo "listening-port=3478
tls-listening-port=5349

external-ip=$4
realm=$3
server-name=$3

lt-cred-mech
fingerprint

userdb=/var/lib/turn/turndb

cert= /etc/letsencrypt/live/$3/cert.pem
pkey= /etc/letsencrypt/live/$3/privkey.pem

use-auth-secret
static-auth-secret=$2

cli-password=$2

log-file=/var/log/turnserver.log
verbose
no-stdout-log"  > /etc/turnserver.conf
```

Y, finalmente, creamos el usuario admin y generamos los certificados con [certbot][certbot]:

```bash
sudo systemctl start coturn

# use certbot to create coturn certificate, you need to have port 80 open to allow the certbot to verify
sudo certbot certonly --standalone --deploy-hook "systemctl restart coturn" -d $3 --agree-tos --no-eff-email --register-unsafely-without-email
```

Este script lo vamos a ejecutar desde una [plantilla de Azure Resource Manager][turnserverdeploy], donde obtendremos el usuario y contraseña a través de parámetros de la plantilla y también la dirección del servidor y su IP desde el despliegue:

```json
"commandToExecute": "[concat('sh installturn.sh ''',parameters('turnAdmin'),''' ''',
  parameters('turnAdminPassword'),''' ''', reference( resourceId('Microsoft.Network/publicIPAddresses',
  variables('ipAddressName')) ).dnsSettings.fqdn, ''' ''',
  reference( resourceId('Microsoft.Network/publicIPAddresses', variables('ipAddressName')) ).ipAddress, '''')]"
```

### Web App

La aplicación web es una modificación [del ejemplo WebRTC][webrtcdemo] para que use el servidor TURN que hemos desplegado. Azure App Service nos permite desplegar una aplicación hecha en Node.js en un servidor Linux, compilarla dentro de un contenedor y ejecutarla automáticamente, usando [Oryx][oryx].

Las modificaciones que he realizado han sido, por una parte generar una credencial única para cada usuario válida por tiempo limitado y la otra ha sido generar la configuración ICE con los valores que obtendremos de la plantilla anterior como son la dirección FQDN del servidor TURN.

```Typescript
if (this.stunServer) {
  if (ice_config.iceServers == null) {
    ice_config.iceServers = [];
  }
  var iceServer: IIceServer = { urls: this.stunServer };
  ice_config.iceServers.push(iceServer);
}
var script: string = data.toString().replace('{{ice_config}}',
                                  util.inspect (ice_config));
```

En la [plantilla de despliegue][webappgit] he añadido la URL del repositorio donde está la aplicación, para que se despliegue automáticamente:

```json
"properties": {
  "RepoUrl": "[parameters('repoURL')]",
  "branch": "[parameters('branch')]",
  "IsManualIntegration": true
}
```

Y el sitio está configurado como un servidor NODE bajo Linux:

```json
"properties": {
    "linuxFxVersion": "NODE|12-lts",
    "webSocketsEnabled": true
}
```

## Despliegue de la solución completa

Usando [múltiples plantillas enlazadas][linkedtemplates] podemos tener obtener valores del despliegue del servidor y pasarlos al despliegue de la aplicación web de una forma bastante limpia. Esto nos permite desplegar en Azure una solución más compleja sin perder la mantenibilidad del sistema con archivos JSON kilométricos.

En este caso tenemos una plantilla principal que lanza las dos plantillas que queremos desplegar, la Máquina Virtual y la Web App. En la segunda nos interesa saber qué DNS tiene la VM para poder configurar el servidor STUN/TURN en la aplicación, así que en la primera añadiremos un valor de salida como este:

```json
"outputs": {
    "dns": {
        "type": "string",
        "value": "[reference( resourceId('Microsoft.Network/publicIPAddresses', variables('ipAddressName')) ).dnsSettings.fqdn]"
    }
}
```

De tal forma que en la plantilla principal podremos usar ese valor como parámetro de la plantilla de la Web App:

```json
"turnServer": { "value": "[reference( resourceId('Microsoft.Resources/deployments', 'turnserverTemplate') ).outputs.dns.value]" }
```

Y así tendremos completamente automatizado nuestro despliegue.

Podéis ver el [ejemplo completo en el repositorio][videochatgit] o bien probar el despliegue directamente con el siguiente botón:

[![Deploy to Azure](http://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fjmservera%2Fvideochat%2Fmain%2Fazuredeploy.json)

## Créditos

* Foto de portada por [Daniel Yanes Arroyo][pinkponk].
* Ejemplo WebRTC de [Mikołaj Wargowski][webrtcdemo]
* Script para despliegue de coturn de [Anastasia Zolochevska][coturnscript]


[motherofalldemos]: https://www.youtube.com/watch?v=M5PgQS3ZBWA&list=PLCGFadV4FqU3flMPLg36d8RFQW65bWsnP
[webrtcdemo]: https://tsh.io/blog/how-to-write-video-chat-app-using-webrtc-and-nodejs/
[STUN]: https://en.wikipedia.org/wiki/STUN
[TURN]: https://en.wikipedia.org/wiki/Traversal_Using_Relays_around_NAT
[ICE]: https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment
[coturngit]: https://github.com/coturn/coturn
[coturn]: https://packages.ubuntu.com/bionic/coturn "paquete para Ubuntu"
[videochatgit]: https://github.com/jmservera/videochat
[webappgit]: https://github.com/jmservera/videochat-webapp
[Azure]: https://azure.microsoft.com
[pinkponk]: https://www.flickr.com/photos/pinkponk/515106921
[certbot]: https://certbot.eff.org/
[turnserverdeploy]: https://github.com/jmservera/videochat/blob/main/turnserver/azuredeploy.json
[coturnscript]: https://devblogs.microsoft.com/cse/2018/01/29/orchestrating-turn-servers-cloud-deployment/
[spherical_cow]: https://en.wikipedia.org/wiki/Spherical_cow
[appservice]: https://azure.microsoft.com/services/app-service/
[aad]: https://azure.microsoft.com/services/active-directory/
[aci]: https://azure.microsoft.com/services/container-instances/
[vm]: https://azure.microsoft.com/services/virtual-machines/
[oryx]: https://github.com/Microsoft/Oryx
[linkedtemplates]: https://docs.microsoft.com/azure/azure-resource-manager/templates/linked-templates
