+++
title =  "Cómo usar Kubernetes para ejecutar código antiguo en Windows (ii)"
date = 2021-09-13T15:29:35+02:00
tags = [ "K8s", "VB6", "legacy" ]
featured_image = "jurassic-park.jpg"
draft = true
+++

En el [capítulo anterior][chapter-i] ensamblamos un contenedor Docker para ejecutar una aplicación servidor escrita en VB6. Hoy vamos a utilizar este contenedor en un clúster de Kubernetes desplegado en Azure usando el servicio AKS.

---

## Capítulo 2: desplegar una aplicación Windows en Kubernetes con AKS

[![El personal técnico viendo pasar los contenedores de VB6. En realidad es una escena de Jurassic Park donde los personajes alucinan viendo a los dinosaurios.][jurassic-park]][jurassic-screenrant]

En esta serie de artículos vamos a ver los siguientes casos:

* [Capítulo 1][chapter-i]: Generar una imagen de contenedor de Windows que ejecute una aplicación VB6.
* [Capítulo 2][chapter-ii]: Desplegar la aplicación en Kubernetes y dar acceso a través de un puerto TCP/IP.
* Capítulo 3: Usar un Ingress Controller para cifrar y enrutar tráfico TCP/IP.
* Capítulo 4: Monitorización de nuestra aplicación a partir de los logs.

[chapter-i]: {{< relref "posts/008-using-k8s-to-run-legacy-vb6-i.md" >}}
[chapter-ii]: {{< ref "posts/008-using-k8s-to-run-legacy-vb6-ii.md" >}}

[jurassic-park]: ./jurassic-park.jpg "El personal técnico viendo pasar los contenedores de VB6. No, es una escena de Jurassic Park de los personajes observando a los dinosaurios, con la misma cara que pone la gente cuando hablas de Kubernetes ejecutando VB6."
[jurassic-screenrant]: https://screenrant.com/jurassic-park-questions-want-answered/ "Fuente original de la imagen - Screenrant"

<!-- [chapter-iii]: < ref "posts/008-using-k8s-to-run-legacy-vb6-iii.md" >}}
[chapter-iv]: < ref "posts/008-using-k8s-to-run-legacy-vb6-iv.md" >}} -->