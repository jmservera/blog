+++
title =  "Cómo usar Kubernetes para ejecutar código antiguo en Windows (ii)"
date = 2021-09-13T15:29:35+02:00
tags = []
featured_image = "jurassic-park.jpg"
description = ""
draft = true
+++

En el [capítulo anterior][chapter-i] ensamblamos un contenedor Docker para ejecutar una aplicación servidor escrita en VB6. Hoy vamos a utilizar este contenedor en un clúster de Kubernetes desplegado en Azure usando el servicio AKS.

---

Recordemos que en esta serie de artículos vamos a ver los siguientes casos:

* [Capítulo 1][chapter-i]: Generar una imagen de contenedor de Windows que ejecute una aplicación VB6.
* [Capítulo 2][chapter-ii]: Desplegar la aplicación en Kubernetes y dar acceso a través de un puerto TCP/IP.
* Capítulo 3: Usar un Ingress Controller para cifrar y enrutar tráfico TCP/IP.
* Capítulo 4: Monitorización de nuestra aplicación a partir de los logs.

## Capítulo 2: desplegar una aplicación Windows en Kubernetes con AKS

[![El personal técnico viendo pasar los contenedores de VB6. En realidad es una escena de Jurassic Park donde los personajes alucinan viendo a los dinosaurios.][jurassic-park]][jurassic-screenrant]

Una vez preparado el contenedor y comprobado que es capaz de servir contenido a través de un puerto TCP/IP, vamos a ver cómo podemos desplegar esa aplicación en Kubernetes y publicar la aplicación al exterior.

Recordemos que estamos trabajando con una aplicación antigua, no está preparada para escalar horizontalmente, ni es un servicio stateless y lo que estamos haciendo es un apaño temporal hasta que modernicemos de verdad la aplicación, pero aún así podremos aprovechar algunas de las ventajas que nos ofrece Kubernetes.

Kubernetes es una solución que se encargará de asegurarse que nuestros contenedores se ejecuten y se comuniquen entre sí en un sistema distribuido. Todo eso lo hará a partir de la descripción que le daremos sobre cómo tiene que ocurrir eso y a partir de ahí se encargará de que todo se mantenga funcionando. Si hay que reiniciar un contenedor porque la aplicación ha dejado de funcionar, o hay que mover la aplicación de servidor porque falta memoria en el que está, Kubernetes se encargará.

En Kubernetes podemos manejar múltiples conjuntos de nodos, que pueden usar diferentes tecnologías. Esto es precisamente lo que vamos a hacer en este caso, en el Azure Kubernetes Service, un servicio gestionado de Kubernetes que nos automatiza la gestión de los nodos donde se ejecutarán los contenedores (en este caso nodo = máquina virtual). Es decir, Kubernetes se encarga de nuestros contenedores, comunicaciones, publicar puertos, balanceo, etc.. y AKS se ocupa de que Kubernetes esté instalado y funcionando en la cantidad de máquinas virtuales que nosotros le indiquemos.

### Desplegar un AKS en Azure

Para poder ejecutar nuestros contenedores en un clúster vamos a desplegar un AKS en una suscripción de Azure, añadirle un conjunto de nodos Windows y describiremos la configuración de los contenedores que se van a ejecutar en el sistema.

Como siempre, en Azure el primer paso es tener un grupo de recursos donde desplegar:

```bash
az group create -g myRG -l northeurope
```

Luego vamos a crear un AKS básico con máquinas pequeñas. Los nodos de sistema serán nodos Linux, luego añadiremos un conjunto de auto-escalado de nodos Windows. Para el tipo de red usaremos Azure CNI, porque la necesitaremos para los nodos Windows: 

```bash
az aks create -g myRG -n myAKS --network-plugin Azure --generate-ssh-keys -s Standard_B4ms
```

Y ahora añadiremos un conjunto de nodos Windows al que se ha creado en el paso anterior:

```bash
az aks nodepool add -g myRG --cluster-name myAKS -s Standard_D2_v2 --os-type Windows --name winnp --node-count 0 --min-count 0 --max-count 3 --enable-cluster-autoscaler --node-taints 'os=windows:NoSchedule' 
```

Hemos marcado a esos nodos con una etiqueta de tipo *[taint][k8s-taint]*, con el valor `os=windows:NoSchedule`, para controlar mejor qué contenedores se ejecutan dentro de esos nodos. Más adelante veremos por qué es útil esa etiqueta, pero básicamente funciona como un repelente de los contenedores que no definan esa etiqueta.

### ¿Cómo subo el contenedor a Kubernetes?

Kubernetes no tiene una interfaz de subida de contenedores, lo que necesitamos es que nuestro clúster tenga acceso al lugar donde guardaremos las imágenes. Podríamos usar, por ejemplo, Docker Hub, que es el lugar de donde hemos estado descargando las imágenes base para generar la nuestra, pero en Azure tenemos el service Azure Container Registry que estará más cerca de nuestro AKS. Así que vamos a crear un registro de imágenes en el mismo grupo de recursos:

```bash
az acr create -g myRG -n myACRjms --admin-enabled true --sku Basic
```

El nombre tiene que ser único porque luego obtendremos una URL para acceder a él, en este caso será `https://myacrjms.azurecr.io`, pero para conectarnos desde Docker necesitaremos unas credenciales:

```bash
az acr credential show --name myacrjms -o table
```

Obtendremos el nombre de usuario y dos credenciales.

```
USERNAME    PASSWORD                          PASSWORD2
----------  --------------------------------  --------------------------------
myacrjms    2ev2AArVrCUCdjGQrLlgxn6fXNX+BdpU  QpOX1t6=1RWXCsxQ2sYJcNpQRDgCmY7I
```








### Definición del Pod

En Kubernetes, un Pod es la unidad más pequeña de despliegue. Dentro de un Pod definiremos lo que necesita esa unidad mínima para poder ejecutarse: podemos tener uno o varias imágenes de contenedor, definir el almacenamiento, límites de memoria o CPU, cómo y en qué orden tienen que arrancar los contenedores, etc. Nosotros vamos a definir en el Pod nuestra imagen en un archivo *yaml* como este:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vbserver-one
  labels:
    app: vbserver-one
    m-app: vbserver
spec:
  nodeSelector:
    "kubernetes.io/os": windows
  containers:
  - name: vbserver-one
    image: jmaks.azurecr.io/vbserver:3.2
    ports:
    - containerPort: 9001
      name: telnet
  tolerations:
  - key: "os"
    operator: "Equal"
    value: "windows"
    effect: "NoSchedule"
```




[chapter-i]: {{< relref "posts/008-using-k8s-to-run-legacy-vb6-i.md" >}}
[chapter-ii]: {{< ref "posts/008-using-k8s-to-run-legacy-vb6-ii.md" >}}

[k8s-taint]: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
[windows-nodes-faq]: https://docs.microsoft.com/azure/aks/windows-faq


[jurassic-park]: ./jurassic-park.jpg "El personal técnico viendo pasar los contenedores de VB6. No, es una escena de Jurassic Park de los personajes observando a los dinosaurios, con la misma cara que pone la gente cuando hablas de Kubernetes ejecutando VB6."
[jurassic-screenrant]: https://screenrant.com/jurassic-park-questions-want-answered/ "Fuente original de la imagen - Screenrant"

<!-- [chapter-iii]: < ref "posts/008-using-k8s-to-run-legacy-vb6-iii.md" >}}
[chapter-iv]: < ref "posts/008-using-k8s-to-run-legacy-vb6-iv.md" >}} -->