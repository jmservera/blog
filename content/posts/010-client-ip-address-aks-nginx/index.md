---
title :  "Obtener la dirección IP del cliente en AKS con .Net Core y NGNIX"
date : 2022-06-17T09:00:00+02:00
tags : ["AKS", "K8s", "Nginx",".Net Core"]
featured_image : "https://live.staticflickr.com/7046/7016112897_8f64ba97e7_k.jpg"
description : ""
draft : false
---
Obtener la IP del cliente es algo imprescindible en muchas aplicaciones, desde
la necesidad de capturar telemetría, o para intentar saber desde qué país
se está conectando el usuario. Sea como sea, si desplegáis una aplicación .Net
Core en AKS y usáis un ingress NGINX, sin ninguna modificación, verás que la
dirección IP del cliente no es la que te esperabas.

<!--more-->

![::ffff:10.240.0.82 esta no es mi IP][wrongip]

Para resolver esto existe una cabecera que debería contener la IP original
llamada `X-Forwarded-For`, pero para que funcione hay que realizar dos cambios.

## Configuración del Ingress Controller

Cuando desplegamos un ingress controller en AKS se suele utilizar `helm` para
instalarlo. En ese caso, necesitaremos añadir una opción que
[está documentada][ingress-basic] pero no viene por defecto:

```bash
--set controller.service.externalTrafficPolicy=Local
```

Como yo suelo usar [Flux][flux] para hacer GitOps en mi cluster, la forma de
indicarlo la podéis ver aquí en la sección values (Flux no admite la forma
abreviada):

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-basic
spec:
  chart:
    spec:
      chart: ingress-nginx
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
  values:
    controller:
      service:
        externalTrafficPolicy: Local
  interval: 1m0s
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
```

Una vez configurado podremos ver que en la cabecera `X-Forwarded-For` aparece ya
 nuestra IP, pero si miramos en la propiedad
```Request.HttpContext.Connection.LocalIpAddress``` seguimos teniendo una IP
que (::ffff:10.240.0.14) que en este caso se corresponde con la IP asignada a mi
 Pod directamente porque estoy usando Azure CNI.

![Vamos mejorando][notmyip]

## Cambios en el código

ASP.Net Core no usa la cabecera `X-Forwarded-For` por defecto, para evitar
problemas de seguridad y porque si ejecutamos la aplicación sin un proxy,
no veremos la IP del cliente. Podemos configurar el middleware Forwarded Headers
para que sí se use esa información. Hay un detalle muy importante a configurar
para que funcione, y es [configurar adecuadamente][aspnetcoreforward] los
`KnownProxies` y las `KnownNetworks`. En el ejmplo oficial simplemente borran la
lista para que funcione:

```csharp
using Microsoft.AspNetCore.HttpOverrides;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages();

if (string.Equals(
    Environment.GetEnvironmentVariable("ASPNETCORE_FORWARDEDHEADERS_ENABLED"),
    "true", StringComparison.OrdinalIgnoreCase))
{
    builder.Services.Configure<ForwardedHeadersOptions>(options =>
    {
        options.ForwardedHeaders = ForwardedHeaders.XForwardedFor |
            ForwardedHeaders.XForwardedProto;
        // Only loopback proxies are allowed by default.
        // Clear that restriction because forwarders are enabled by explicit 
        // configuration.
        options.KnownNetworks.Clear();
        options.KnownProxies.Clear();
    });
}

var app = builder.Build();

app.UseForwardedHeaders();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseAuthorization();

app.MapRazorPages();

app.Run();
```

Aquí habría que poner en esa lista los valores de la red que tenemos en K8s, que
podemos pasar por variable de entorno, pero es importante fijarse en cómo vienen
esas direcciones pues a veces vendrán en formato IPv4 (10.240.0.1) y otras en
IPv6 (::ffff:10.240.0.1). Es importante que pongamos la que está llegando a
nuestra aplicación (en los pantallazos veréis que en mi caso están en formato
IPv6).

Después de aplicar estos cambios en el código podremos finalmente recibir la IP
del cliente dentro del :

![Esta sí que es mi IP][yesmyip]

## Créditos

* Foto de portada por [B Klug][hackerpic].

<!--links-->
[aspnetcoreforward]: https://docs.microsoft.com/aspnet/core/host-and-deploy/proxy-load-balancer?view=aspnetcore-6.0#forward-the-scheme-for-linux-and-non-iis-reverse-proxies
[flux]: https://fluxcd.io/
[ingress-basic]: https://docs.microsoft.com/azure/aks/ingress-basic?tabs=azure-cli#create-an-ingress-controller
<!--pics-->
[hackerpic]: https://www.flickr.com/photos/bklug/7016112897/in/photolist-bFZpVK-GepVqk-4cQ9xp-2kGuN8H-2kGuN8c-2kGyYH1-2kGyYJU-2kGyoJZ-2kGyYCm-2kGyYCM-2kGyYG4-8dPtP-4FERTh-2kGuN7R-3s54x-62EU2k-GRg6YN-2kGyoPi-2kGuN5m-3RULze-bFZqyB-2kGyYDZ-2epJJU-9fLSnv-FU5crS-8oDdGs-7F1uv6-7F1xnT-7Ufa6-nse15Y-7oATva-o5Bd3s-8Z1gcc-5TdAqq-5fE2p9-v3dgFs-4q9N5q-9RaRoA-4q9MMh-4q9Mpo-2mbJcvN-4t7VGS-ojQUAZ-4XbwhY-59jidK-2mbGLsY-4doPb7-KLbwz-21g5T-24wn8tc "Mysterious Hacker by B Klug"
[wrongip]: ./wrongip.png "::ffff:10.240.0.82 esta no es mi IP"
[notmyip]: ./notmyip.jpeg "Vamos mejorando"
[yesmyip]: ./yesmyip.jpeg "Esta sí que es mi IP"
