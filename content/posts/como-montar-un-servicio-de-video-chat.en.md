+++
title =  "How to create a video chat service"
date = 2020-06-24T10:25:22+02:00
tags = ["Azure","WebRTC","TURN","PoC","ARM Templates"]
featured_image = "https://live.staticflickr.com/207/515106921_a83de99560_o.jpg"
description = "If Doug could in 1968 you can"
draft = false
+++

I believe I live in modern times until I realize that in the '60s you could already [buy a videoconferencing system][pinkponk], and you could meet Douglas Engelbart in a [mind-blowing demo in 1968][motherofalldemos].
![Douglas Engelbart performance in "The mother of all demos"](/como-montar-videochat/engelbartdemo.png "Doug's shopping list demo")
So, now that we have advanced in the XXI century, how hard should it be to create a private video chat using a serverless WebRTC app, just like Western Electric could have announced sixty years ago? As usual, it's not completely easy, and you need to overcome some problems to do the P2P connection.

## WebRTC

Nowadays, inside our browser we have the technology that allows us to create a communication channel between two remote browsers, i.e., we can send data between two client applications without the need of a server, under [ideal conditions][spherical_cow]. For example, we could build a simple solution like the one seen in the post [WebRTC and Node.js][webrtcdemo] and we could have a rather functional video chat app. After following the guide and [updating some libraries][videochatgit], it will work if you use it in your local network, but once you test it in unrelated networks over the internet you will have a lot of trouble to connect, and the application will probably fail.

[![India, Delhi - Indian-style cable spaghetti - February 2018 by Cyprien Hauser](https://live.staticflickr.com/65535/49204696853_6df9abbc5c_c.jpg)](https://flic.kr/p/2hY3W7i "Where's Wally wire edition?")

What's happening here is that between the two distant points there will be probably a bunch of firewalls, NAT, proxies, and lots of other arcane network stuff that prevents you from having a direct connection between the two browsers. There are lots of written words about why and how this works, so I let you some bibliography about the problems and their solutions below:

* [Session Traversal Utilities for NAT (STUN)][STUN], it's the toolset we need to allow two remote clients to meet through the Real World&trade; network mess.
* [Traversal Using Relays around NAT (TURN)][TURN], when STUN is not enough, you need a proxy to stream the video between the two points.
* [Interactive Connectivity Establishment (ICE)][ICE], it's the way WebRTC has to decide whether STUN or TURN should be used.

## Show me the code

> If you prefer to read JSON instead of this boring text, check the Github repos directly: the [deployment template][videochatgit] and [the web app][webappgit]

As the articles above explain, it looks like we will need a streaming server after all. Even though it is only needed when the two browsers cannot establish a direct connection,
you usually do a video call with people out of your local network and most of the time we will need to use a TURN server. The good news is that it already exists an OSS project that deploys a TURN / STUN / ICE server called [coturn][coturngit], and it's also available as an [Ubuntu package][coturn]; the bad news is that for production environments this will be expensive, as you will need a big and scalable infrastructure to maintain it.

To deploy all these parts in Azure is relatively easy: the WebRTC web app will be deployed in the [Azure App Service][appservice], which has high availability built-in, SSL, and integrates with [Azure Active Directory][aad] that we will use to manage the access to the service in a later post. The TURN server could be deployed in an [Azure Container Instance][aci], but we need to use UDP and TCP at the same time and this is still not supported in ACI, so we will use a [Virtual Machine][vm] with a startup script that will install the TURN server.

### TURN Server

The first thing we are going to build is the Virtual Machine for the TURN server, with a startup script to configure the *coturn* application and will generate SSL certificates for it. The first part of the script installs the needed runtimes:

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

Next, we generate a configuration file, using some parameters that we will send to the script from our deployment system (see the parameters above):

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

And, in the end, we create the admin user and generate the SSL certificates with [certbot][certbot]:

```bash
sudo systemctl start coturn

# use certbot to create coturn certificate, you need to have port 80 open to allow the certbot to verify
sudo certbot certonly --standalone --deploy-hook "systemctl restart coturn" -d $3 --agree-tos --no-eff-email --register-unsafely-without-email
```

This script will run from an [Azure Resource Manager template][turnserverdeploy], where we will ask for the user and password through the template parameters, and we will also get the full server name and IP address from the deployment:

```json
"commandToExecute": "[concat('sh installturn.sh ''',parameters('turnAdmin'),''' ''',
  parameters('turnAdminPassword'),''' ''', reference( resourceId('Microsoft.Network/publicIPAddresses',
  variables('ipAddressName')) ).dnsSettings.fqdn, ''' ''',
  reference( resourceId('Microsoft.Network/publicIPAddresses', variables('ipAddressName')) ).ipAddress, '''')]"
```

### Web App

The web app is a customized one based on [the WebRTC example][webrtcdemo] to make it the TURN server we have deployed in the first step. Azure App Service has the ability to automatically compile and execute a Node.js inside a Linux server, using [Oryx][oryx] behind the scenes.

The customizations I made were one to create unique credentials for each user for a limited time to increase security, and the second was to generate the ICE configuration with the values we get from the template, such as FQDN of the TURN server.

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

So, in the [deployment template][webappgit], I added the URL for the web app repo, this will make the App Service get it from Github and then compile the full project into a container and run it:

```json
"properties": {
  "RepoUrl": "[parameters('repoURL')]",
  "branch": "[parameters('branch')]",
  "IsManualIntegration": true
}
```

The site is configured as a NODE server using Linux:

```json
"properties": {
    "linuxFxVersion": "NODE|12-lts",
    "webSocketsEnabled": true
}
```

## Full deployment

Using [multiple linked templates][linkedtemplates] we can get the values from the TURN server deployment and pass them to the web app with a rather clean structure. This allows us to deploy in Azure very complex solutions without losing the maintainability of the system with VeryLargeJSON&reg; files.

For this case, I have created a main template that launches another two templates for the resources I need to deploy: the virtual machine and the web app. In this second one, we need to know the DNS name that has been created for the virtual machine, so we can configure ICE. In the first template, we need to add something like this at the end:

```json
"outputs": {
    "dns": {
        "type": "string",
        "value": "[reference( resourceId('Microsoft.Network/publicIPAddresses', variables('ipAddressName')) ).dnsSettings.fqdn]"
    }
}
```

This way, in the main template we can gather this value and pass it as a parameter to the second one containing the web app:

```json
"turnServer": { "value": "[reference( resourceId('Microsoft.Resources/deployments', 'turnserverTemplate') ).outputs.dns.value]" }
```

And this is the way we can fully automate the deployment of a complete solution.

You can see the [full example in the git repo][videochatgit] and test it directly in your Azure subscription by clicking the button below:

[![Deploy to Azure](https://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fjmservera%2Fvideochat%2Fmain%2Fazuredeploy.json)

## Credits

* Main picture [Daniel Yanes Arroyo][pinkponk].
* WebRTC example from [Mikołaj Wargowski][webrtcdemo]
* Original *coturn* deployment script by [Anastasia Zolochevska][coturnscript]

[motherofalldemos]: https://www.youtube.com/watch?v=M5PgQS3ZBWA&list=PLCGFadV4FqU3flMPLg36d8RFQW65bWsnP
[webrtcdemo]: https://tsh.io/blog/how-to-write-video-chat-app-using-webrtc-and-nodejs/
[STUN]: https://en.wikipedia.org/wiki/STUN
[TURN]: https://en.wikipedia.org/wiki/Traversal_Using_Relays_around_NAT
[ICE]: https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment
[coturngit]: https://github.com/coturn/coturn
[coturn]: https://packages.ubuntu.com/bionic/coturn "Ubuntu package"
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