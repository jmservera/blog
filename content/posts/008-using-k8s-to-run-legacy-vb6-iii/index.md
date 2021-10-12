+++
title =  "Cómo usar Kubernetes para ejecutar código antiguo en Windows (III)"
date = 2021-10-12T20:44:12+02:00
tags = [ "K8s", "VB6", "legacy" ]
featured_image = "puertasalcampo.jpg"
draft = true
+++

Si has leído los capítulos anteriores sabrás que en el [capítulo 2][chapter-ii] desplegamos un contenedor Windows en Kubernetes con un servicio TCP/IP muy sencillo. Publicar un servicio en una IP pública en Kubernetes fue fácil y funcionó, pero eso no lo convierte en un sistema preparado para producción. Tendremos que preocuparnos de que el tráfico esté cifrado, ser capaces de servir a muchos clientes distintos, monitorizar el servicio, actualizarlo, etc.
 
En este capítulo nos encargaremos de los dos primeros puntos mediante un servicio de control de acceso.

<!--more-->

Recordemos el contenido de esta serie de artículos:

* [Capítulo 1][chapter-i]: Generar una imagen de contenedor de Windows que ejecute una aplicación VB6.
* [Capítulo 2][chapter-ii]: Desplegar la aplicación en Kubernetes y dar acceso público a través de un puerto TCP/IP.
* [Capítulo 3][chapter-iii]: Usar un Ingress Controller para cifrar y enrutar tráfico TCP/IP.
* Capítulo 4: Monitorización de nuestra aplicación en Kubernetes a partir de los logs.
* Capítulo 5: Gestionar y actualizar montones de contenedores.

## Capítulo 3: Usar un Ingress Controller para cifrar y enrutar tráfico TCP/IP

Ahora que ya podemos publicar el servicio, necesitaremos que los usuarios de nuestros clientes puedan encontrar el mismo para conectarse. Recordemos que estamos modernizando un servicio antiguo y con mucha deuda técnica, así que tenemos algunas limitaciones. La más importante es que para cada cliente necesitamos una configuración inicial diferente, así que necesitaremos una instancia para cada uno. Esto significa que los usuarios tendrán que conectarse a una instancia diferente dependiendo de a qué cliente pertenezcan.

### ¿Y si publico múltiples servicios?

En el capítulo anterior creamos un servicio que nos proporcionaba una IP pública para nuestra aplicación. Una forma sencilla de tener una instancia diferente para cada grupo de usuarios es crear un servicio por cada grupo. Eso nos generará tantas IPs públicas como servicios publiquemos. Pero este planteamiento presenta algunos problemas a nivel de costes, pues cada IP y cada regla del Load Balancer tienen coste, no es mucho pero multiplicado por cientos de clientes acaba siendo una suma considerable.

### ¿Qué hace el Ingress Controller?

Un Ingress Controller, o controlador de entrada, nos proporciona enrutamiento configurable, TLS y un proxy inverso para los servicios externos que publicamos de Kubernetes. Uno de los más famosos es el basado en [NGINX][nginx] y también tenemos un controlador para el [Azure Application Gateway][azure-app-gateway]. La mayoría de estos *ingress* enrutan tráfico http y https, y suelen estar preparados para crear rutas basadas en dns o en la url, lo que conocemos por enrutado de capa 7.

En nuestro caso, el servicio trabaja en la capa 4 así que no podremos usar esos controladores de entrada, pero hay alguno como [Kong][kong-ingress] que nos permitirá enrutar tráfico TCP/IP, siempre y cuando usemos una conexión TLS. El motivo de esto último es que si no usamos TLS, la dirección DNS no vendrá dentro del paquete, porque se resuelve antes y la conexión se realiza directamente a una dirección IP.

https://docs.konghq.com/kubernetes-ingress-controller/2.0.x/guides/using-tcpingress/

## Enlaces

* [Azure Kubernetes Service][aks]
* El código de ejemplo en [GitHub][ghcode]
* Instalación del [kubectl][kubectl]
* Qué es un [pod][pods]
* [Taints & Tolerations][k8s-taint]

[chapter-i]: {{< relref "posts/008-using-k8s-to-run-legacy-vb6-i.md" >}}
[chapter-ii]: {{< ref "posts/008-using-k8s-to-run-legacy-vb6-ii.md" >}}
[chapter-iii]: {{< ref "posts/008-using-k8s-to-run-legacy-vb6-iii.md" >}}

[azcli]: https://docs.microsoft.com/cli/azure/install-azure-cli
[azure-app-gateway]: https://kubernetes.github.io/ingress-nginx/deploy/
[aks]: https://docs.microsoft.com/azure/aks/intro-kubernetes
[kong-ingress]: https://docs.konghq.com/kubernetes-ingress-controller/
[kubectl]: https://kubernetes.io/docs/tasks/tools/
[ingress]: https://kubernetes.io/docs/concepts/services-networking/ingress/
[nginx]: https://kubernetes.github.io/ingress-nginx/deploy/

[jurassic-park]: ./jurassic-park.jpg "El personal técnico viendo pasar los contenedores de VB6. No, es una escena de Jurassic Park de los personajes observando a los dinosaurios, con la misma cara que pone la gente cuando hablas de Kubernetes ejecutando VB6."
[jurassic-screenrant]: https://screenrant.com/jurassic-park-questions-want-answered/ "Fuente original de la imagen - Screenrant"
