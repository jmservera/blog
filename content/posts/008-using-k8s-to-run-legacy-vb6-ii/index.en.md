+++
title =  "How to use Kubernetes to modernize Windows Applications (II)"
date = 2021-10-05T21:30:35+02:00
tags = [ "K8s", "VB6", "legacy" ]
featured_image = "jurassic-park.jpg"
draft = false
+++

In the [previous chapter][chapter-i] we assembled a Docker container to execute a service written in VB6. Today, we are going to use this container in a Kubernetes cluster deployed in Azure using the Azure Kubernetes Service.

<!--more-->

Let's remember, in this post series we are covering these topics:

* [Chapter 1][chapter-i]: Generate a Windows Container Image to run a [VB6 server application](#vb6server).
* [Chapter 2][chapter-ii]: Run the app in a Kubernetes cluster and provide a connection to the TCP/IP port.
* Chapter 3: Use an Ingress Controller to provide communication encryption and route the TCP/IP to the right container.
* Chapter 4: Application monitorization from the logs.
* Chapter 5: Manage and update a bunch of containers.

## Chapter 2: Run the app in a Kubernetes cluster and provide a connection to the TCP/IP port

[![IT crowd observing the VB6 containers fly by. Ok, right, actually it's a scene from the Jurassic Park film where the characters freak out when they see the dinosaurs.][jurassic-park]][jurassic-screenrant]

Once the container is built and we checked that it serves content through a TPC/IP port, we will see how to deploy it in a Kubernetes cluster, and how to use a public IP address to connect to it.

Just remember that we are working with a legacy app that it is not prepared for horizontal scaling, neither is a stateless service, and we are creating a temporary fix until we could modernize this app. Said this, we can take advantage of the benefits that Kubernetes has for us.

### How will Kubernetes help?

First of all, a little disclaimer. We must do some introspection and assess what other options we have before deciding that Kubernetes is the best one. 
Lo primero que nos tenemos que preguntar a la hora de pensar si debemos usar Kubernetes es qué otras opciones hay. Muchas veces nuestra aplicación la podremos desplegar en algún sistema FaaS, PaaS, CaaS, IaaS o incluso directamente en Hierro&trade;, pues probablemente será más sencillo que usar Kubernetes. En el caso concreto que estoy comentado, la aplicación estaba desplegada en IaaS, pero usaba cientos de VMs que eran muy difíciles de mantener, administrar y actualizar, aquí Kubernetes nos va ayudar mucho tanto en los costes como en el mantenimiento de la aplicación, así que es justificable el trabajo adicional.

Kubernetes es una solución que se asegurará que nuestros contenedores se ejecuten y se comuniquen entre sí en un sistema distribuido. Todo eso lo hará a partir de la descripción que le daremos sobre cómo tiene que ocurrir eso y a partir de ahí se encargará de que todo se mantenga en funcionamiento. Si hay que reiniciar un contenedor porque la aplicación ha dejado de funcionar, o hay que mover la aplicación de servidor porque falta memoria en el que está, o necesitamos actualizar cientos o miles de pods sin tener que realizar montones de pasos manuales o scripts muy complejos, Kubernetes se encargará de hacer ese trabajo.

En este caso vamos a utilizar el Azure Kubernetes Service ([AKS][aks]), un servicio gestionado de Kubernetes que nos automatiza la gestión de los nodos donde se ejecutarán los contenedores (en este caso nodo = máquina virtual). Es decir, Kubernetes se encarga de nuestros contenedores, comunicaciones, publicar puertos, balanceo, etc.. y AKS se ocupa de que Kubernetes esté instalado y funcionando en la cantidad de máquinas virtuales que nosotros le indiquemos, además de proporcionarnos otros servicios de Azure como el balanceo de carga externo, IP pública, Gateway, etc.

### Desplegar un AKS en Azure

Para poder ejecutar nuestros contenedores vamos a desplegar un [Azure Kubernetes Service][aks], le añadiremos un conjunto de nodos Windows, porque los nodos principales tienen que ser Linux, y describiremos la configuración de los contenedores que se van a ejecutar en el sistema.

Como siempre en Azure, el primer paso es tener un grupo de recursos donde desplegar, desde `PowerShell` (porque estamos usando Windows Containers) creamos el grupo de recursos con la línea de comando [Az][azcli]:

```ps1
$rgName="myRG"
$region="northeurope"

az group create -g $rgName -l $region
```

Si todavía no lo tenemos, vamos a necesitar un registro de contenedores donde publicar las imágenes y que nuestro clúster tenga acceso al mismo.

```ps1
$aksName="myAKS"
$acrName="${aksName}ACR"

az acr create -g $rgName -n $acrName --admin-enabled true --sku Basic
```

Luego vamos a crear un AKS básico con máquinas pequeñas conectado a ese registro. Los nodos de sistema tienen que ser nodos Linux, pero luego añadiremos un conjunto de auto-escalado de nodos Windows. Para el tipo de red usaremos Azure CNI, porque la necesitaremos para los nodos Windows: 

```ps1
az aks create -g $rgName -n $aksName --network-plugin Azure --generate-ssh-keys -s Standard_B4ms --attach-acr $acrName
```

Y ahora añadiremos un conjunto de nodos Windows al que se ha creado en el paso anterior:

```ps1
az aks nodepool add -g $rgName --cluster-name $aksName -s Standard_D2_v2 --os-type Windows --name winnp --node-count 0 --min-count 0 --max-count 3 --enable-cluster-autoscaler --node-taints 'os=windows:NoSchedule' 
```

Hemos marcado a esos nodos con una etiqueta de tipo *[taint][k8s-taint]*, con el valor `os=windows:NoSchedule`, para controlar mejor qué contenedores se ejecutan dentro de esos nodos. Más adelante veremos por qué es útil esa etiqueta, pero básicamente funciona como un repelente de los contenedores que no definan esa etiqueta.

### ¿Cómo subo el contenedor a Kubernetes?

Kubernetes no tiene una interfaz de subida de contenedores, lo que necesitamos es que nuestro clúster tenga acceso al lugar donde guardaremos las imágenes. Podríamos usar, por ejemplo, Docker Hub, que es el lugar de donde hemos estado descargando las imágenes base para generar la nuestra, pero en Azure tenemos el service Azure Container Registry que estará más cerca de nuestro AKS y será nuestro registro privado en el mismo grupo de recursos. Es el que hemos creado antes con el comando `az acr`.

Para conectarnos al registro de contenedores desde Docker necesitaremos unas credenciales, para conseguirlas podemos ejecutar el siguiente comando en PowerShell:

```ps1
$acrCreds=(az acr credential show --name $acrName --query [username,passwords[0].value] | ConvertFrom-Json)

docker login "${acrName}.azurecr.io" -u $acrCreds[0] -p $acrCreds[1]
```

Finalmente, tenemos que etiquetar nuestro contenedor y subirlo a ese registro, usamos el contenedor `vb6` que creamos en el capítulo anterior:

```bash
docker tag vb6 "${acrName}.azurecr.io/vb6:1.0"
docker push "${acrName}.azurecr.io/vb6:1.0"
```

<details>
  <summary>Haz click aquí para ver el script completo hasta ahora...</summary>

```ps1
# PowerShell

$rgName="myRG"
$aksName="myAKS"
$acrName="${aksName}ACR"
$region="northeurope"

az group create -g $rgName -l northeurope

# Container registry creation and upload
az acr create -g $rgName -n $acrName --admin-enabled true --sku Basic
$acrCreds=(az acr credential show --name $acrName --query [username,passwords[0].value] | ConvertFrom-Json)

docker login "${acrName}.azurecr.io" -u $acrCreds[0] -p $acrCreds[1]
docker tag vb6 "${acrName}.azurecr.io/vb6:1.0"
docker push "${acrName}.azurecr.io/vb6:1.0"

# AKS creation
az aks create -g $rgName -n $aksName --network-plugin Azure --generate-ssh-keys -s Standard_B4ms --attach-acr $acrName

az aks nodepool add -g $rgName --cluster-name $aksName -s Standard_D2_v2 --os-type Windows --name winnp --node-count 0 --min-count 0 --max-count 3 --enable-cluster-autoscaler --node-taints 'os=windows:NoSchedule'

```
</details>


### Definición de elementos de Kubernetes

Para poder conectar a nuestro clúster, necesitaremos instalar el comando [`kubectl`][kubectl] y obtener las credenciales de conexión:

```ps1
az aks get-credentials -n $aksName -g $rgName --admin
```

En Kubernetes, la unidad más pequeña de despliegue es un [Pod][pods]. Dentro de un Pod definiremos lo que necesita esa unidad mínima para poder ejecutarse; podemos tener uno o varios contenedores, definir el almacenamiento, límites de memoria o CPU, cómo y en qué orden tienen que arrancar los contenedores, etc.

Tenemos múltiples formas de crear elementos en kubernetes, podemos ejecutar comandos directamente con la línea de comando `kubectl` para ejecutar un contenedor:

```bash
kubectl run vbserver --image=myaksacr.azurecr.io/vb6:1.0
```

Pero también podemos usar una definición en `yaml` o `json` que podremos almacenar en un repositorio de código y tenerlo versionado. Nosotros vamos a definir en el Pod nuestra imagen en un archivo `yaml` como este:

```yaml
# vbserverpod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: vbserver
  labels:
    app: vbserver # Etiqueta para identificar el Pod
spec:
  nodeSelector:
    "kubernetes.io/os": windows
  containers:
  - name: vbserver-container
    image: jmaks.azurecr.io/vb6:1.0
    ports:
    - containerPort: 9001
      name: telnet
  tolerations:
  - key: "os"
    operator: "Equal"
    value: "windows"
    effect: "NoSchedule"
```

En este pod hemos definido un contenedor con nuestra imagen, que indica el puerto por el que sirve el contenido. Hemos añadido esa `toleration` que habíamos creado antes en los nodos Windows, para asegurarnos que el Pod podrá ejecutarse en ese tipo de nodos, e indicado con el `nodeSelector` que solo se ejecuta en los nodos Windows. La toleration nos sirve para evitar que se desplieguen pods con contenedores Linux en esos nodos, ya que el nodeSelector sólo es efectivo si está definido, pero como no es obligatorio, los pods se intentarán desplegar en el primer nodo que esté libre. Hacerlo así nos evitará problemas futuros de pods que no se levantan porque no existe una imagen Windows para ellos.

Para desplegarlo podemos usar el comando apply que puede leer ese `yaml` y aplicar los cambios necesarios en el clúster:

```ps1
kubectl apply -f vbserverpod.yaml
```

Al desplegar un pod en Kubernetes, el sistema intentará descargar el contenedor y ejecutarlo, pero no podremos acceder a él porque sólo estará disponible en la red interna con una dirección IP efímera, esto es, si se reinicia el pod la IP se pierde. Si queremos probarlo, podemos utilizar el comando `kubectl port-forward pod/vbserver 9001` , eso nos creará un proxy local para acceder a nuestro pod.

También podremos ver en qué estado está nuestro pod con el comando: 

```ps1
kubectl describe pod vbserver
```

Y ver los logs con:

```ps1
kubectl logs vbserver
```

Para poder publicar nuestro pod hacia el exterior necesitaremos definir un servicio, un elemento virtual de Kubernetes que nos permite proporcionar una IP a un conjunto de pods en la red interna, a nivel de nodo o pública. La diferencia con la IP del pod es que ésta no cambia por mucho que reiniciemos o destruyamos los pods, mientras dure la definición del servicio tendremos esa IP, además hará un balanceo de carga simple entre todos los pods que se encuentren bajo el selector definido.

Usaremos la siguiente línea de comando para crear el servicio directamente (añadimos el modificador -o yaml para ver la definición como resultado): 

```ps1
kubectl expose pod/vbserver --name vbserver-service --port 9001 --target-port 9001 --type LoadBalancer -o yaml
``` 

Para obtener una IP pública usamos un servicio de tipo LoadBalancer, que en AKS se corresponderá con un Load Balancer Standard de Azure y publicará en una IP el puerto de nuestros nodos, el `yaml` generado por la operación será parecido a este:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: vbserver
  name: vbserver-service
spec:
  ports:
  - port: 9001
    protocol: TCP
    targetPort: 9001
  selector:
    app: vbserver # este selector es la misma etiqueta que hemos definido en el Pod más arriba
  type: LoadBalancer
```

La forma que tiene el servicio para encontrar los pods a los que tiene que dirigir el tráfico de esa IP es a través del selector, así, si tuviéramos múltiples pods, el balanceador iría distribuyendo la carga entre las instancias de los pods que estén funcionando y tengan esa etiqueta. En Kubernetes podemos crear pods que escalen automáticamente mediante los [deployments][deployment] y el [Horizontal Pod Autoscaler][hpa], pero eso lo dejaremos para más adelante.

Si ahora listamos los servicios en Kubernetes, veremos que se ha asignado una IP pública a nuestro servicio además de la IP interna que tiene en el cluster:

```bash
> kubectl get services -o wide

NAME                          TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)          AGE
service/kubernetes            ClusterIP      10.0.0.1     <none>          443/TCP          69m
service/vbserver              LoadBalancer   10.0.98.53   20.67.149.184   9001:32691/TCP   17m

```

Y así ya tenemos nuestro servicio publicado en ese puerto a través de un Load Balancer de Azure, si vamos al portal podremos ver en el grupo de recursos autogenerado por AKS, que ahí hay un Load Balancer y se ha creado una regla tanto en el LB como en el NSG para servir a través de ese puerto.

Así que podremos ejecutar un telnet a esa dirección para conectar con nuestro servidor, pero esta vez ya la tenemos en una IP pública en Internet:

```bash 
> telnet 20.67.149.184 9001 
Connected to server: vbserver
```

## Enlaces

* [Azure Kubernetes Service][aks]
* El código de ejemplo en [GitHub][ghcode]
* Instalación del [kubectl][kubectl]
* Qué es un [pod][pods]
* [Taints & Tolerations][k8s-taint]

[chapter-i]: {{< relref "posts/008-using-k8s-to-run-legacy-vb6-i.md" >}}
[chapter-ii]: {{< ref "posts/008-using-k8s-to-run-legacy-vb6-ii.md" >}}

[azcli]: https://docs.microsoft.com/cli/azure/install-azure-cli
[aks]: https://docs.microsoft.com/azure/aks/intro-kubernetes
[deployment]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
[ghcode]: https://github.com/jmservera/legacyvb6ink8s
[hpa]:  https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
[k8s-taint]: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
[kubectl]: https://kubernetes.io/docs/tasks/tools/
[pods]: https://kubernetes.io/docs/concepts/workloads/pods/
[windows-nodes-faq]: https://docs.microsoft.com/azure/aks/windows-faq


[jurassic-park]: ./jurassic-park.jpg "El personal técnico viendo pasar los contenedores de VB6. No, es una escena de Jurassic Park de los personajes observando a los dinosaurios, con la misma cara que pone la gente cuando hablas de Kubernetes ejecutando VB6."
[jurassic-screenrant]: https://screenrant.com/jurassic-park-questions-want-answered/ "Fuente original de la imagen - Screenrant"

<!-- [chapter-iii]: < ref "posts/008-using-k8s-to-run-legacy-vb6-iii.md" >}}
[chapter-iv]: < ref "posts/008-using-k8s-to-run-legacy-vb6-iv.md" >}} -->