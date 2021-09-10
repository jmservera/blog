+++
title= "Cómo usar Kubernetes para modernizar aplicaciones Windows (i)"
subtitle= "Generar una imagen de contenedor de Windows que ejecute una aplicación VB6"
date= 2021-09-10T11:00:47+02:00
featured_image="/008-using-k8s-to-run/vb6.png"
tags = [ "K8s", "VB6", "legacy" ]
draft= false
+++

Vamos a ver en una serie de 4 capítulos cómo podemos aprovechar las prácticas más modernas de contenedores para sacar el último aliento a esas aplicaciones obsoletas que quizá tengamos ejecutándose en nuestros sistemas.

Si hacemos un inventario de las aplicaciones servidor que tenemos en nuestros servidores Windows, en los mejores casos serán aplicaciones web ejecutándose en un IIS que podríamos migrar con alguna [herramienta semi-automática][azure-migrate], pero en muchos otros casos no será tan fácil. Hace poco me encontré con el caso de un servidor TCP/IP escrito en VB6 que se desplegaba en unos cuantos cientos de máquinas virtuales. A priori, parece que para modernizar esta aplicación a un entorno de contenedores tendremos que reescribir el código. Seguramente, desde el punto de vista de desarrollo y mantenimiento de la solución, sería la mejor opción, pero todo es cuestión de encontrar los compromisos adecuados entre lo que nos exige el negocio y la capacidad que tenemos en el equipo. En este caso, tener un paso intermedio en el que podemos ahorrarnos montones de máquinas virtuales, desplegando en su lugar contenedores es una buena opción, y así damos más tiempo al equipo de desarrollo para que pueda volver a escribir toda la lógica de esa aplicación.

---

## Capítulo 1: Generar una imagen de contenedor de Windows para ejecutar VB6

Hoy en día, Visual Basic 6 ya no es un lenguaje popular y a nadie se le ocurriría usarlo para un nuevo desarrollo. Pero la realidad ahí fuera es que todavía hay muchas aplicaciones en producción usando VB6, y siguen apareciendo preguntas nuevas en [Stack Overflow][so-vb6]. Tanto es así que, aunque el IDE esté descatalogado desde 2008, seguimos dando soporte al runtime en modo ["It Just Works"][vb6-support], incluso en Windows Server 2019 y Windows 10.

Si combinamos esto con el uso de contenedores Windows y el despliegue de esos contenedores en Kubernetes, vamos a poder ejecutar las aplicaciones VB6 sin necesidad de montar máquinas virtuales para cada aplicación.

> **Nota:** es importante recordar que no podemos ejecutar un escritorio remoto en un contenedor de Windows, así que esta técnica se limita a las aplicaciones en modo servidor, sin interfaz de usuario.

En esta serie de artículos vamos a ver los siguientes casos:

* [Capítulo 1][chapter-i]: Generar una imagen de contenedor de Windows que ejecute una [aplicación VB6](#vb6server)
* Capítulo 2: Desplegar la aplicación en Kubernetes y dar acceso a través de un puerto TCP/IP
* Capítulo 3: Usar un Ingress Controller para cifrar y enrutar tráfico TCP/IP
* Capítulo 4: Monitorización de nuestra aplicación a partir de los logs

## Aplicación de ejemplo VB6

No tiene mucho sentido a estas alturas de la historia de VB6 ponernos a explicar demasiado el código de ejemplo. Encontraréis el código en la carpeta `source` del [repositorio de ejemplo en GitHub][ghcode]. A continuación tenéis algunos detalles básicos para entender qué hace la aplicación y cómo nos conectaremos a ella.

### Código del servidor VB6 {#vb6server}

Para demostrar que el sistema funciona y probar que puedo instalar algunos componentes OCX, he creado una aplicación de prueba que puede escuchar vía `Telnet` en el puerto `9001` y responde con el nombre del servidor:

```vb
tcpServer(intMax).Accept requestid
' SEND SERVER INFO TO CLIENT
tcpServer(intMax).SendData ("Connected to server: " + tcpServer(intMax).LocalHostName + vbCrLf)
``` 

### Empaquetado de la aplicación VB6

Como necesitamos que sea una aplicación de consola, vamos a generar el ejecutable por línea de comando:

```bash
"C:\Program Files (x86)\Microsoft Visual Studio\VB98\VB6.EXE" /MAKE vb6app.vbp /OUTDIR ..\app
"C:\Program Files (x86)\Microsoft Visual Studio\VB98\LINK.EXE" /EDIT /SUBSYSTEM:CONSOLE ..\app\myapp.exe
```

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
[vb6-support]: https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-basic-6/visual-basic-6-support-policy
[so-vb6]: https://stackoverflow.com/questions/tagged/vb6

[chapter-i]: {{< relref "#aplicación-de-ejemplo-vb6" >}}


[telnet-connected]: /008-using-k8s-to-run/telnet-connected.png "Telnet terminal"