+++
title= "How to use Kubernetes to modernize Windows Applications (I)"
date= 2021-09-20T11:00:47+02:00
featured_image="vb6.png"
tags = [ "K8s", "VB6", "legacy" ]
draft= false
+++

Here it is, the first in a series of 4 posts on how to make something new from old Windows Server Applications that you may have running on-premises wasting a lot of hardware and energy.

When you do an assessment of running applications in your Windows servers, as a best scenario, you may find some old web apps running on an IIS server that could be migrated using a [semi-automated tool][azure-migrate], but, in my experience, there are lots of not-so-edge cases that won't be as easy as that. Recently, I stumbled on a TCP/IP server application written in VB6 that was running in a few hundred virtual machines. At first, you may think that this application does not deserve the effort and it would be better to do a complete rewrite. Well, on the one side, from a developer or operator point of view you are completely right, but, on the other side, from a business perspective it's all about trade-offs, and you may not have the budget to build a new app from scratch. So, what if you could reduce the current solution cost by containerizing it, so you save in hardware, and use these savings to push for the rewrite of a new and modern solution? Let's talk about it.

---

## Chapter 1: Generate a Windows Container image to run VB6

Is this even possible? Visual Basic 6 is not the popular programming language it used to be anymore, and nobody would use it for new developments. But, the reality out there is that there are still a lot of legacy VB6 applications running in production systems; as proof of this, you can still find people asking new questions on [Stack Overflow][so-vb6]. It is to such an extent that, even though the IDE has been retired since 2008, we still provide support for the runtime in 
["It Just Works"][vb6-support] mode up to Windows Server 2019 and Windows 10.

This means we can create a Windows Container image that will run VB6 applications, and we can use this image to run VB6 applications in a Kubernetes cluster, without any changes to the application itself and without the hassle of maintaining hundreds of Windows Server machines separately (but remember you still have to [do some updates and patches][patch-and-upgrade]).

> **Side note:** remember that containers are headless machines, so we won't be able to run a Remote Desktop session or any graphical interface action in these Windows Containers.

In this series I'm going to explain how to:

* [Chapter 1][chapter-i]: Generate a Windows Container Image to run a [VB6 server application](#vb6server)
* [Chapter 2][chapter-ii]: Run the app in a Kubernetes cluster and provide a connection to the TCP/IP port.
* Chapter 3: Use an Ingress Controller to provide communication encryption and route the TCP/IP to the right container.
* Chapter 4: Application monitorization from the logs.

## A VB6 example application

![Splash Screen de VB6][vb6-splash]

I'm going to use some very simple example code, but there's no sense in doing a deep explanation of what it does. You can find the code in the folder named `source` from [the GitHub repo][ghcode]. Let me explain some basics below.

### VB6 Server Code {#vb6server}

The app is a simple TCP/IP server to check that our system is running the app, it is serving content, and to demonstrate that you can use old OCX components. So, you can communicate with this app via `Telnet` on the `9001` port. The app will answer with the server name upon connection:

```vb
tcpServer(intMax).Accept requestid
' SEND SERVER INFO TO CLIENT
tcpServer(intMax).SendData ("Connected to server: " + tcpServer(intMax).LocalHostName + vbCrLf)
``` 

### Packaging the VB6 app

We will use an old trick to force the app to run in console mode, using this command line:

```bash
"C:\Program Files (x86)\Microsoft Visual Studio\VB98\VB6.EXE" /MAKE vb6app.vbp /OUTDIR ..\app
"C:\Program Files (x86)\Microsoft Visual Studio\VB98\LINK.EXE" /EDIT /SUBSYSTEM:CONSOLE ..\app\myapp.exe
```

Once we get the application, we can start generating the container image with the application and all the needed components.

## Windows Container Image creation

To create a container image, first, you need to create a `Dockerfile` file. This file will contain the list of commands and instructions to build the image as an unattended process, that is, you won't be able to interact with the install process, but once created the image will be completely prepared to run.

In the case of a VB6 application, the install steps are as simple as copying the runtime to the application folder and registering the needed components. For this one, I'm using an ActiveX component that is not included in the standard Windows distribution, so I need to copy it to the application folder and register it with the `regsvr32` command. There are some additional libraries that do not need registering, it is enough to copy them to the application folder. You can find in the [repo][ghcode] these ones:

* msvbvm60.dll: this is the VB6 runtime library.
* MSWINSCK.OCX: the ActiveX component used for the TCP/IP server; this one needs to be registered.
* VBA6.DLL: the Visual Basic for Applications library, used to write messages to the console.
* myapp.exe: this is the compiled app from the `source` folder, so you don't need to install the VB6 IDE on your machine.

So, we will write in the `Dockerfile` the steps to prepare and install our application. The file begins indicating that we will use the (\`) as an escape character, to avoid problems with the backslash (\\) used as folder separator in Windows.
The next one indicates the Windows version for the container. As we pretend to run it inside Kubernetes, we will use the `servercore ltsc2019` version, the one compatible with the [Azure Kubernetes Service (AKS)][aks].

```docker
# escape=`
FROM mcr.microsoft.com/windows/servercore:ltsc2019
```

The following ones create the folder for our application and copy and register the ActiveX component:

```docker
RUN md c:\app
WORKDIR c:\app

COPY .\app\MSWINSCK.OCX C:\APP
RUN regsvr32 /s c:\app\MSWINSCK.OCX
```

And, finally, we indicate the port, copy all the application files, and indicate the command that will run once our container starts:

```docker
EXPOSE 9001

COPY .\app\* c:\app\

CMD ["myapp.exe"]
```

> Note: I copy and register the MSWINSCK.OCX before all the other ones, even though they reside in the same folder, and then I copy the rest of the files in a later action. This has a negative impact on the number of layers our container will have, but it comes with an advantage because as the OCX installation takes a lot of time and this usually does not change, but the application may change often, we can use the Docker layer to avoid repeating the install process each time we make a change in the application. Docker will use a cached copy to speed up the build process.

<details>
  <summary>And here's the complete `Dockerfile` ...</summary>

```docker
# escape=`
FROM mcr.microsoft.com/windows/servercore:ltsc2019

RUN md c:\app
WORKDIR c:\app

COPY .\app\MSWINSCK.OCX C:\APP
RUN regsvr32 /s c:\app\MSWINSCK.OCX

EXPOSE 9001

COPY .\app\* c:\app\

CMD ["myapp.exe"]
```

</details>

Once we have the `Dockerfile`, we can build the image using the following command:

```cmd
docker build -t vb6 .
```

We can test it by running it and publishing its port:

```cmd
docker run -d -p 9001:9001 vb6
```

And once it is running we can connect to it using a `telnet` client:

```cmd
telnet localhost 9001
```

If everything is ok, we should see something similar to the following message, with the container id as the server name:

![Telnet terminal with the text 'Connected to server: c433ff27eb8f'][telnet-connected]

This id, which we also saw with `docker run`, will be useful to debug and manage our container. Docker will also generate a random name that we can use for this same purpose, but it is easier with the id, because you can use the first two or three digits to execute some commands in your container instance. In our case, the id is `c433ff27...`, and we can run any of these commands:

```bash
docker logs c4 -f
docker exec -it c4 cmd
docker kill c4
```

The first one will show live logs of our container, the second one will open an interactive command prompt in the container, and the last one will kill the container.

## Continue playing

We made it easier to fit an old application into our DevOps workflow. We are generating a container image that is easier to move and deploy than a VM image. In the next article, we will see how to deploy it to Kubernetes, and take advantage of all the benefits that it offers, like the ability to automatically restart the container if it crashes.

## Links

* Example code in [GitHub][ghcode]
* You will need [Docker Desktop][docker-desktop] to generate the container image for Windows.

[aks]: https://azure.microsoft.com/services/kubernetes-service
[azure-migrate]: https://azure.microsoft.com/services/azure-migrate/
[docker-desktop]: https://www.docker.com/products/docker-desktop
[ghcode]: https://github.com/jmservera/legacyvb6ink8s
[patch-and-upgrade]: https://docs.microsoft.com/azure/architecture/operator-guides/aks/aks-upgrade-practices
[so-vb6]: https://stackoverflow.com/questions/tagged/vb6
[vb6-support]: https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-basic-6/visual-basic-6-support-policy

[chapter-i]: {{< relref "#aplicación-de-ejemplo-vb6" >}}
[chapter-ii]: {{< ref "posts/008-using-k8s-to-run-legacy-vb6-ii" >}}

[telnet-connected]: telnet-connected.png "Telnet terminal"
[vb6-splash]: vb6.png "VB6 Splash Screen"