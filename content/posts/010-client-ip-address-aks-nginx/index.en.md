---
title :  "Get the client IP address in AKS with .Net Core and NGNIX"
date : 2022-06-17T09:00:00+02:00
tags : ["AKS", "K8s", "Nginx",".Net Core"]
featured_image : "https://live.staticflickr.com/7046/7016112897_8f64ba97e7_k.jpg"
description : ""
draft : false
---
There are many times when you need to get your client IP address: for
application telemetry, for getting the client country or any other info you may
need. Anyway, when you deploy a .Net Core app on AKS and you are using an NGINX
ingress, that you didn't explicitely configure, the IP address you will be
receiving into your app may not be what you expected.

<!--more-->

![::ffff:10.240.0.82 this is not my IP][wrongip]

To fix this, there's the `X-Forwarded-For` header, that should contain the
original IP address, but for this to work you need to change two different
things.

## Ingress Controller configuration

When deploying the ingress controller with `helm` on AKS, you need to set an
specific option; it is [well documented][ingress-basic] but it is not
activated by default:

```bash
--set controller.service.externalTrafficPolicy=Local
```

As in my case I use [Flux][flux], so I can do GitOps on my cluster, I need to
define the this configuration setting inside the `values` section (Flux does not
admit the abbreviated form):

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

Once configured, we can already see our IP in the `X-Forwarded-For` header, but
when watching the
```Request.HttpContext.Connection.LocalIpAddress``` property, it still has the
::ffff:10.240.0.14 value, that, in this particular case, as I'm using Azure CNI,
it comes from the assigned IP of the Pod that is currently running.

![This is better][notmyip]

## Code changes

ASP.Net Core does not use the `X-Forwarded-For` header by default. This helps
you avoid security risks, and furthermore, if the application is not behind a
proxy you wouldn't see the real IP address. In our case, we can configure the
Forwarded Headers middleware to correctly use the header. There's an important
detail to configure to make it work: you need to
[properly configure][aspnetcoreforward] the `KnownProxies` and `KnownNetworks`.
Otherwise it will not work. In the official example the list is cleared:

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

But, for the sake of security, I would preferrably put the actual values we have
in our K8s cluster. We can pass these values as environment variables, but it is
really important to notice in which format they are coming, because sometimes
we will get an IPv4 address format (e.g: '10.240.0.1') and in other
configurations we may have an IPv6 address format (e.g: '::ffff:10.240.0.1').

After applying these changes you can see how we finally got the actual client
address IP:

![Yes, this is my real IP][yesmyip]

## Credits

* Cover picture by [B Klug][hackerpic].

<!--links-->
[aspnetcoreforward]: https://docs.microsoft.com/aspnet/core/host-and-deploy/proxy-load-balancer?view=aspnetcore-6.0#forward-the-scheme-for-linux-and-non-iis-reverse-proxies
[flux]: https://fluxcd.io/
[ingress-basic]: https://docs.microsoft.com/azure/aks/ingress-basic?tabs=azure-cli#create-an-ingress-controller
<!--pics-->
[hackerpic]: https://www.flickr.com/photos/bklug/7016112897/in/photolist-bFZpVK-GepVqk-4cQ9xp-2kGuN8H-2kGuN8c-2kGyYH1-2kGyYJU-2kGyoJZ-2kGyYCm-2kGyYCM-2kGyYG4-8dPtP-4FERTh-2kGuN7R-3s54x-62EU2k-GRg6YN-2kGyoPi-2kGuN5m-3RULze-bFZqyB-2kGyYDZ-2epJJU-9fLSnv-FU5crS-8oDdGs-7F1uv6-7F1xnT-7Ufa6-nse15Y-7oATva-o5Bd3s-8Z1gcc-5TdAqq-5fE2p9-v3dgFs-4q9N5q-9RaRoA-4q9MMh-4q9Mpo-2mbJcvN-4t7VGS-ojQUAZ-4XbwhY-59jidK-2mbGLsY-4doPb7-KLbwz-21g5T-24wn8tc "Mysterious Hacker by B Klug"
[wrongip]: ./wrongip.png "::ffff:10.240.0.82 not my IP"
[notmyip]: ./notmyip.jpeg "There's my IP somwhere"
[yesmyip]: ./yesmyip.jpeg "Yes my IP"
