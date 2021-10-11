+++
title =  "How to use Kubernetes to modernize Windows Applications (II)"
date = 2021-10-11T10:45:45+02:00
tags = [ "K8s", "VB6", "legacy" ]
featured_image = "jurassic-park.jpg"
draft = false
+++

In the [previous chapter][chapter-i], we assembled a Docker container to execute a service written in VB6. Today, we are going to use this container in a Kubernetes cluster deployed in Azure using the Azure Kubernetes Service.

<!--more-->

Let's remember, in this post series we are covering these topics:

* [Chapter 1][chapter-i]: Generate a Windows Container Image to run a [VB6 server application](#vb6server).
* [Chapter 2][chapter-ii]: Run the app in a Kubernetes cluster and provide a public connection to the TCP/IP port.
* Chapter 3: Use an Ingress Controller to provide communication encryption and route the TCP/IP to the right container.
* Chapter 4: Application logging and monitorization in Kubernetes.
* Chapter 5: Manage and update a bunch of containers.

## Chapter 2: Run the app in a Kubernetes cluster and provide a connection to the TCP/IP port

[![Our IT crew observing the VB6 containers fly by. Well, actually it's a scene from the Jurassic Park film where the characters freak out when they see the dinosaurs. And I feel like the old man in that picture.][jurassic-park]][jurassic-screenrant]

Once the container is built and we checked that it serves content through a TPC/IP port, it is time to deploy it in a Kubernetes cluster and find out how to use a public IP address to connect to it.

Just remember that we are working with a legacy app that is not scale-out ready, nor is it a stateless service, and we are creating a temporary fix until we could modernize this app. Said this, we can take advantage of the benefits that Kubernetes has for us.

### How will Kubernetes help?

Before we go further I want to warn you with a small disclaimer. It is important to assess what are the other options we have before deciding that Kubernetes is the best one. Many times, our app could be deployed on FaaS, PaaS, CaaS, IaaS, or even Bare Metal&trade;, because, probably, any of these systems would be easier to manage than a Kubernetes cluster. In this particular case, the app is already deployed on IaaS, but managing and updating hundreds of VMs is a very difficult and error-prone job, so the additional setup work in Kubernetes is completely justified here.

Kubernetes will ensure our containers execute and communicate properly inside a distributed and scalable system. All this work will be done from a static description of how all this has to be managed. We will write this description in a source-controlled file, and the service will ensure that the current configuration complies with the description and everything works as expected. If there's a need to restart a container because the application stopped working, or a container must be moved to another server instance because it ran out of resources, o we need to update hundreds or thousands of images, Kubernetes will do this for us.

We are going to use the Azure Kubernetes Service ([AKS][aks]), a managed Kubernetes service that automates the management of the nodes (in Azure the Virtual Machines) where the containers will run. Kubernetes will manage our containers, networking, ports, load balancing, and so on, and the AKS service will ensure that Kubernetes is properly installed and runs the number of nodes we indicated. It will also provide us with other important services from Azure, like external load balancing, gateways, public IPs and node scale-out.

### Deploy an AKS in Azure

To execute our containers we will deploy an [Azure Kubernetes Service][aks], and we will add a Windows node pool were to run our containers.

As usual, the first step is creating a resource group in our Azure subscription. As we are using Windows, let's stick to `PowerShell` (but the steps are very similar with a `bash` command-line). Using the [Az][azcli] run these commands:

```ps1
$rgName="myRG"
$region="northeurope"

az group create -g $rgName -l $region
```

If we still don't have one, we will need a container registry to publish our container images so our cluster can reach them:

```ps1
$aksName="myAKS"
$acrName="${aksName}ACR"

az acr create -g $rgName -n $acrName --admin-enabled true --sku Basic
```

Then we will create a basic AKS with small VMs, connected to the registry we created before. The system nodes need to be Linux, but we will add a Windows scale set later on. For the networking driver we will choose Azure CNI because it is mandatory for Windows nodes, and the last parameter connects our cluster to the container registry:

```ps1
az aks create -g $rgName -n $aksName --network-plugin Azure --generate-ssh-keys -s Standard_B4ms --attach-acr $acrName
```

Now that we have a cluster, we can add some Windows nodes to it:

```ps1
az aks nodepool add -g $rgName --cluster-name $aksName -s Standard_D2_v2 --os-type Windows --name winnp --node-count 0 --min-count 0 --max-count 3 --enable-cluster-autoscaler --node-taints 'os=windows:NoSchedule' 
```

Notice that we have labeled the nodes with a *[taint][k8s-taint]* with the value `os=windows:NoSchedule`. This label allows us to have better control over which containers will execute into these nodes. We will see later why this taint label is useful, but shortly, it works as a container repellant for the ones that don't define this label.

### And now, how do I upload my container to Kubernetes?

Well, Kubernetes does not have an interface for uploading containers, what we need is to have access to a container registry where the images are stored. We could use a public one, like Docker Hub, the place where we found the base Windows images in the previous chapter, but we also have the Azure Container Registry that will be closer to our AKS and will be our private registry. This is the one we created before with the `az acr` command, and we attached it to our cluster during the cluster creation.

We need to download the registry credentials so we can use them from the `docker` command-line:

```ps1
$acrCreds=(az acr credential show --name $acrName --query [username,passwords[0].value] | ConvertFrom-Json)

docker login "${acrName}.azurecr.io" -u $acrCreds[0] -p $acrCreds[1]
```

And to upload our image to the registry we need to tag it with our registry name and upload it with the cli. We will use the `vb6` image we created in the previous chapter:

```bash
docker tag vb6 "${acrName}.azurecr.io/vb6:1.0"
docker push "${acrName}.azurecr.io/vb6:1.0"
```

<details>
  <summary>Click here to see the complete script...</summary>

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


### Kubernetes definition files

Now we need to tell Kubernetes what we want to execute there. To connect and manage our cluster we have to install the [`kubectl`][kubectl] command-line (install it from the link above), and we also need the credentials. This command will download an API key used by `kubectl` to connect to our cluster:

```ps1
az aks get-credentials -n $aksName -g $rgName --admin
```

In Kubernetes, the smallest unit you can deploy is a [Pod][pods]. Inside the Pod, we will define what this minimal unit needs to execute; we can have one or many containers, define the storage, CPU, and memory limits, how and in which order our containers must run, and many other things.

There are many ways to create items in Kubernetes. We can, for example, execute a direct order to execute a container with the `kubectl` cli:

```ps1
kubectl run vbserver --image=myaksacr.azurecr.io/vb6:1.0
```

But the preferred way is to create a `yaml` or `json` definition that we will store in a code repository, so we have a version-controlled infra as code repo. We will define our Pod in a `yaml` file like this one:

```yaml
# vbserverpod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: vbserver
  labels:
    app: vbserver # Label to identify the Pod
spec:
  nodeSelector:
    "Kubernetes.io/os": windows
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

In this pod, we have defined a container with our custom image and indicated the port we are using. We have added a `toleration` that matches the `taint` we defined in the Windows nodes to ensure that the only pods that run there are the ones we want. We have also indicated with the `nodeSelector` that this Pod can only execute in Windows nodes. The `toleration` avoids potential problems, because usually Pods that run on Linux do not specify a nodeSelector and this would cause many problems because they may try to run inside a Windows node and we will get lots of weird errors.

To deploy this `yaml` inside our cluster we can run this command:

```ps1
kubectl apply -f vbserverpod.yaml
```

When deploying a pod, Kubernetes will download the container image and execute it inside the cluster. But, we won't be able to access it, because it will only be available inside the internal network with an ephemeral IP, that is, if the pod restarts we lose this IP address. We still can test it using a port forwarding trick executing the `kubectl port-forward pod/vbserver 9001` command, that will create a local proxy so we can reach our pod.

Other useful commands are:

* `describe` that will show us what's happening with our pod: 
  ```ps1
  kubectl describe pod vbserver
  ```
* An if the pod is running, we can see its `logs`:
  ```ps1
  kubectl logs vbserver
  ```

### Obtaining a public IP address for our pod

To have access to the pod from outside the cluster we need to expose it with a `service`, a virtual Kubernetes element that allows us to provide an IP address to a set of pods. This IP can be one of the internal network, so other pods can reach it, but can also be published at node level or we can obtain a public IP to the outside world. The main difference with the internal IP of the pod is that this address will not change with pod restarts, and we will also get a simple load balancer between all the pods under the same defined selector.

We will use this command-line to create the service, adding the `-o yaml` modifier to see the result of the command: 

```ps1
kubectl expose pod/vbserver --name vbserver-service --port 9001 --target-port 9001 --type LoadBalancer -o yaml
``` 

As we asked for a LoadBalancer, it will create a public IP address that in AKS will come from a Standard Load Balancer in Azure, the generated `yaml` file will look like this:

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
    app: vbserver # this selector uses the same label/value pair we defined in the Pod before
  type: LoadBalancer
```

The created service finds the pods to redirect the incoming traffic using the provided selector, so, if we had many pods, the load balancer would distribute the traffic among all the working instances that had the same label defined in the selector. Within Kubernetes we can create pods that scale automatically using the [deployments][deployment] and the [Horizontal Pod Autoscaler][hpa], but as our current service cannot scale-out we won't use this feature by now.

Now we can list the services and we will see the public IP address of the service and the internal IP used inside the cluster:

```bash
> kubectl get services -o wide

NAME                          TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)          AGE
service/Kubernetes            ClusterIP      10.0.0.1     <none>          443/TCP          69m
service/vbserver              LoadBalancer   10.0.98.53   20.67.149.184   9001:32691/TCP   17m

```

And voilà, we now have our service published in a public IP and port through the Azure Load Balancer. If you go to the Azure portal and dig into the autogenerated resource group for AKS, you will see the Load Balancer and how a rule has been automatically created there and in the Network Security Group to allow traffic through this port.

Now we can run a `telnet` command towards this address to check that our service answers the connection, but this time we will use the public IP address:

```bash 
> telnet 20.67.149.184 9001 
Connected to server: vbserver
```

## To be continued

In the next chapter, we will cover different techniques to run multiple instances of this service, so we can serve multiple customers with a different configuration each. See you soon.

## Useful links

* [Azure Kubernetes Service][aks] explained.
* All this code can be found in my [GitHub][ghcode] repo.
* You will need to install the [kubectl][kubectl] command-line.
* What are [pods][pods]?
* How to use [Taints & Tolerations][k8s-taint].

[chapter-i]: {{< relref "posts/008-using-k8s-to-run-legacy-vb6-i.md" >}}
[chapter-ii]: {{< ref "posts/008-using-k8s-to-run-legacy-vb6-ii.md" >}}

[azcli]: https://docs.microsoft.com/cli/azure/install-azure-cli
[aks]: https://docs.microsoft.com/azure/aks/intro-Kubernetes
[deployment]: https://Kubernetes.io/docs/concepts/workloads/controllers/deployment/
[ghcode]: https://github.com/jmservera/legacyvb6ink8s
[hpa]:  https://Kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
[k8s-taint]: https://Kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
[kubectl]: https://Kubernetes.io/docs/tasks/tools/
[pods]: https://Kubernetes.io/docs/concepts/workloads/pods/
[windows-nodes-faq]: https://docs.microsoft.com/azure/aks/windows-faq


[jurassic-park]: ./jurassic-park.jpg "Our IT crew observing the VB6 containers fly by. Well, actually it's a scene from the Jurassic Park film where the characters freak out when they see the dinosaurs, with the same face that people put when you talk about Kubernetes running VB6."
[jurassic-screenrant]: https://screenrant.com/jurassic-park-questions-want-answered/ "Fuente original de la imagen - Screenrant"

<!-- [chapter-iii]: < ref "posts/008-using-k8s-to-run-legacy-vb6-iii.md" >}}
[chapter-iv]: < ref "posts/008-using-k8s-to-run-legacy-vb6-iv.md" >}} -->