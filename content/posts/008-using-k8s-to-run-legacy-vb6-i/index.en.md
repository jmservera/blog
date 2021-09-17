+++
title= "How to use Kubernetes to modernize Windows Applications (i)"
date= 2021-09-15T11:00:47+02:00
featured_image="vb6.png"
tags = [ "K8s", "VB6", "legacy" ]
draft= true
+++

And here we are with the first in a series of 4 posts on how to make something new from old Windows Server Applications, that you may have running on-premises wasting a lot of hardware and energy.

When you do an assessment of running applications in your Windows servers, in the best scenario you may find some old web apps running on an IIS server that could be migrated using some [semi-automated tool][azure-migrate], but, in my experience, there are lots of not-so-edge cases that won't be as easy. Recently, I stumbled on a TCP/IP server application written in VB6 that was running in a few hundred virtual machines. At first, you may think that this application does not deserve the effort and it would be better to do a complete rewrite. Well, on the one side, from a developer or operator point of view you are completely right, but, on the other side, from a business perspective it's all about trade-offs, and you may not have the budget to build a new app from scratch. So, what if you could reduce the current solution cost by containerizing it, so you save in hardware, and use these savings to push for the rewrite of a new and modern solution? Let's talk about it.

---

## Chapter 1: Generate a Windows Container image to run VB6

Is this even possible? Visual Basic 6 is not the popular programming language it used to be anymore, and nobody would use it for new developments. But, the reality out there is that there are still a lot of legacy VB6 applications running in production systems; as proof, you can still find people asking new questions on [Stack Overflow][so-vb6]. It is to such an extent that, even the IDE has been retired since 2008, we still provide support for the runtime in 
["It Just Works"][vb6-support] mode up to Windows Server 2019 and Windows 10.

This means we can create a Windows Container image that will run VB6 applications, and we can use this image to run VB6 applications in a Kubernetes cluster, without any changes to the application itself and without the hassle of maintaining hundreds of Windows Server machines separately (but remember you still have to [patch them][patch-and-upgrade]).

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

Once we get the application, we can start generating the container image. I'm using some additional components that we will add
Y una vez tenemos la aplicación ya podemos empezar a generar el contenedor, como es una aplicación VB6 y usamos algunos componentes adicionales, tendremos que tener acceso a algunos archivos `.ocx` y `.dll` que deberán acompañar a la aplicación.

## Generación del contenedor Windows

La generación de una imagen de contenedor requiere que describamos todos los pasos de instalación de la aplicación en un archivo `Dockerfile`, que ejecutará todos esos pasos de forma desatendida. Es decir, no podemos depender de una instalación que necesite interacción con el usuario.

En el caso de nuestra aplicación VB6, la instalación es bastante sencilla pues bastará con copiar el runtime en la misma carpeta de la aplicación y registrar los componentes adicionales que utiliza. Para esta aplicación utilizo un componente en forma de control ActiveX que se registra con el comando `regsvr32`, y algunas librerías adicionales que no necesitan registrarse, es suficiente con que estén en la misma carpeta de la aplicación. Veréis en el [repositorio][ghcode] que he adjuntado las siguientes:

* msvbvm60.dll: esta librería contiene el runtime de VB6.
* MSWINSCK.OCX: el componente ActiveX para crear el servidor TCP/IP. Necesita registrarse.
* VBA6.DLL: la librería de Visual Basic for Applications que utilizo para poder escribir a la consola.
* myapp.exe: la aplicación compilada desde la carpeta `source`, así no tenéis que instalaros el entorno de VB6 para compilarla.

Lo que vamos a hacer es indicar en un `Dockerfile` los pasos para que nuestra aplicación quede instalada y preparada para ejecutarse en el contenedor. Empezaremos por indicar que el carácter de escape va a ser la tilde invertida (\`). De esta forma evitaremos tener que estar utilizando caracteres de escape para la barra invertida (\\), que es la que usa Windows como separador para el path.
En la siguiente línea le indicaremos la versión de Windows del contenedor. Como más adelante queremos ejecutarlo en Kubernetes, vamos a usar la versión `servercore ltsc2019` que es la que es compatible con el servicio [Azure Kubernetes Service (AKS)][aks].

```docker
# escape=`
FROM mcr.microsoft.com/windows/servercore:ltsc2019
```

En el siguiente paso vamos a crear una carpeta donde tendremos nuestra aplicación y copiaremos e instalaremos el componente ActiveX que necesitamos:

```docker
RUN md c:\app
WORKDIR c:\app

COPY .\app\MSWINSCK.OCX C:\APP
RUN regsvr32 /s c:\app\MSWINSCK.OCX
```

Y, para acabar, indicamos el puerto que está publicando nuestra aplicación, copiamos todo el resto de archivos que necesita nuestra aplicación e indicamos el comando a ejecutar al arrancar el contenedor:

```docker
EXPOSE 9001

COPY .\app\* c:\app\

CMD ["myapp.exe"]
```

> Nota: veréis que estoy copiando MSWINSCK.OCX de forma individual y luego copio el resto de archivos, aunque estén todos en la misma carpeta. Hacerlo así tiene una ventaja a la hora de desarrollar, pues el sistema de construcción de imágenes de Docker almacena en caché cada capa, para aprovecharla en el caso de que nada haya cambiado. El componente OCX seguro que no cambiará y, como registrar el componente ActiveX es un poco lento, nos evitaremos ese paso cada vez que cambiemos algo en la aplicación.

<details>
  <summary>Aquí podéis ver el  `Dockerfile` completo...</summary>

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

Una vez tengamos el Dockerfile completo podemos generar la imagen:

```cmd
docker build -t vb6 .
```

Y luego ejecutarlo para comprobar que está todo funcionando, publicamos el puerto para poder conectarnos directamente:

```cmd
docker run -d -p 9001:9001 vb6
```

Una vez tengamos el contenedor en marcha podremos conectar al mismo utilizando el comando telnet de la siguiente forma: 

```cmd
telnet localhost 9001
```

Y deberíamos ver en el terminal el nombre de nuestro servidor, que en este caso se corresponderá con el identificador del contenedor:

![Telnet terminal with the text 'Connected to server: c433ff27eb8f'][telnet-connected]

Ese identificador, que también habremos visto al ejecutar `docker run`, nos será útil a la hora de depurar y gestionar el contenedor. Aunque Docker genera un nombre aleatorio con el que también podremos gestionarlo, es mucho más fácil usar los primeros dígitos del identificador para ejecutar comandos de docker. En nuestro caso el identificador es `c433ff27...`, si tenemos pocos contenedores ejecutándose es poco probable que tengamos una colisión de los primeros dígitos del identificador, así que podemos escribir alguno de los siguientes comandos:

```bash
docker logs c4 -f
docker exec -it c4 cmd
docker kill c4
```

El primer comando nos servirá para ver los logs del contenedor en tiempo real, el segundo para obtener una línea de comando interactiva dentro del contenedor y el último para parar el contenedor.

## Sigue jugando

Ahora ya podemos meter esa aplicación antigua dentro de nuestro ciclo de DevOps. Estamos generando una imagen de contenedor que es más fácil de mover y desplegar que una imagen de máquina virtual. En el próximo capítulo de la serie veremos cómo podemos desplegar este contenedor en Kubernetes para poder aprovechar las ventajas que nos ofrece, como por ejemplo que nuestra aplicación vuelva a levantarse automáticamente en caso de fallo.

## Enlaces

* El código de ejemplo en [GitHub][ghcode]
* Para generar el contendor hace falta [Docker Desktop][docker-desktop], configurado para contenedores Windows

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