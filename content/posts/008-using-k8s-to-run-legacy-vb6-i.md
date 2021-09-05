+++
title= "Cómo usar Kubernetes para ejecutar código antiguo en Windows (I)"
date= 2021-08-27T23:12:47+02:00
draft= true
tags = [ "K8s", "VB6", "legacy" ]
+++

Vamos a ver en una serie de capítulos cómo podemos aprovechar las prácticas más modernas de contenedores para sacarle el último aliento a esas aplicaciones obsoletas que quizá tengamos ejecutándose en nuestros sistemas.

Migrar desarrollos antiguos de aplicaciones servidor en Windows a entornos más modernos puede ser un auténtico quebradero de cabeza. En los mejores casos será una aplicación web ejecutándose en un IIS que podríamos migrar con alguna herramienta automática, pero en muchos otros casos no será tan fácil; hace poco me encontré con el caso de un servidor TCP/IP escrito en VB6. A priori parece que para modernizar esta aplicación a un entorno de contenedores, tendremos que reescribir el código. Seguramente, desde el punto de vista de desarrollo y mantenimiento de la solución, sería la mejor opción, pero todo es cuestión de encontrar los compromisos adecuados entre lo que nos exige el negocio y la capacidad que tenemos en el equipo. En este caso, tener un paso intermedio en el que podemos ahorrarnos montones de máquinas virtuales desplegando en su lugar contenedores era una buena opción, y así damos más tiempo al equipo de desarrollo para que pueda volver a escribir toda la lógica de esa aplicación.

---

Hoy en día, Visual Basic 6 ya no es un lenguaje popular y a nadie se le ocurriría usarlo para un nuevo desarrollo. Pero la realidad ahí fuera es que todavía hay muchas aplicaciones en producción usando VB6. Tanto es así que, aunque el IDE esté descatalogado desde 2008, seguimos dando soporte al rutime en modo ["It Just Works"](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-basic-6/visual-basic-6-support-policy) hasta Windows Server 2019 y Windows 10.

Si combinamos esto con el uso de contenedores Windows y el despliegue de esos contenedores en Kubernetes, vamos a poder ejecutar las aplicaciones VB6 sin necesidad de montar máquinas virtuales para cada aplicación.

> **Nota:** es importante recordar que no podemos ejecutar un escritorio remoto en un contenedor de Windows, así que esta técnica se limita a las aplicaciones en modo servidor, sin interfaz de usuario.

En esta serie de artículos vamos a ver cómo:

* [Capítulo 1][chapter-i]: Generar una imagen de contenedor de Windows con una [aplicación VB6](#vb6server)
* [Capítulo 2][chapter-ii]: Desplegar la aplicación en Kubernetes y dar acceso a través de un puerto TCP/IP
* Usar un Ingress Controller para cifrar y enrutar tráfico TCP/IP
* Cómo podemos realizar la monitorización de nuestra aplicación

## Aplicación de ejemplo VB6

No tiene mucho sentido a estas alturas de la historia de VB6 ponernos a explicar demasiado el código de ejemplo. Está todo el código en la carpeta `source` del [repositorio de ejemplo en GitHub][ghcode]. A continuación tenéis algunos detalles.

### Código del servidor VB6 {#vb6server}

Para demostrar que funciona y probar que puedo instalar algunos componentes OCX, he creado una aplicación de prueba que puede escuchar y reponder vía `Telnet` en el puerto `9001` y responde con el nombre del servidor:

```vb
tcpServer(intMax).Accept requestid
' SEND SERVER INFO TO CLIENT
tcpServer(intMax).SendData ("Connected to server: " + tcpServer(intMax).LocalHostName + vbCrLf)
``` 
Y devuelve la respuesta OK de vez en cuando al cliente:

```vb
If Data = vbCr Or Data = vbLf Or Data = vbCrLf Then
        tcpServer(index).SendData ("OK" + vbCrLf)
End If
```

Una aplicación sencilla para poder probar que nuestro contenedor responde.

### Empaquetado de la aplicación VB6

Como necesitamos que sea una aplicación de consola, vamos a generar el ejecutable por línea de comando:

```bash
"C:\Program Files (x86)\Microsoft Visual Studio\VB98\VB6.EXE" /MAKE vb6app.vbp /OUTDIR ..\app
"C:\Program Files (x86)\Microsoft Visual Studio\VB98\LINK.EXE" /EDIT /SUBSYSTEM:CONSOLE ..\app\myapp.exe
```

Y una vez tenemos la aplicación ya podemos empezar a generar el contenedor, como es una aplicación VB6 y usamos algunos componentes adicionales, tendremos que tener acceso a algunos archivos `.ocx` y `.dll` que deberán acompañar a la aplicación.

## Generación del contenedor Windows

Como os comentaba al principio, el runtime de VB6 es compatible con Windows Server 2019, pero eso no quiere decir que esté instalado. La buena noticia es que basta con copiar el runtime en la misma carpeta de la aplicación. Además, necesitaremos instalar o registrar los componentes adicionales que utilice nuestra aplicación.

Para esta aplicación utilizo un componente en forma de control ActiveX que se registra con el comando `regsvr32`, y algunas librerías adicionales que no necesitan registrarse, es suficiente con que estén en la misma carpeta de la aplicación. Veréis en el [repositorio][ghcode] que ahí están:

* msvbvm60.dll: esta librería contiene el runtime de VB6
* MSWINSCK.OCX: el componente ActiveX que necesita registrarse 
* VBA6.DLL: la librería de Visual Basic for Applications, que utilizo para poder escribir a la consola
* myapp.exe: la aplicación compilada desde la carpeta `source`, así no tenéis que instalaros el entorno de VB6 para compilarla.

Lo que vamos a hacer es indicar en un `Dockerfile` los pasos para que nuestra aplicación quede instalada y preparada para ejecutarse en el contenedor. Empezaremos por indicar que el carácter de escape va a ser la tilde invertida (\`). De esta forma evitaremos tener que estar utilizando carácteres de escape para la barra invertida (\\), que es la que usa windows como separador para el path.
En la siguiente línea le indicaremos la versión de Windows del contenedor. Como más adelante queremos ejecutarlo en Kubernetes, vamos a usar la versión `servercore ltsc2019`.

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

> Nota: estoy copiando primero MSWINSCK.OCX y luego el resto de archivos en lugar de hacerlo todo de golpe. Hacerlo así tiene una ventaja a la hora de desarrollar, pues el sistema de construcción de imagenes de Docker almacena en caché cada capa. Como registrar el componente ActiveX es un poco lento, nos evitaremos ese paso cada vez que cambiemos algo en la aplicación.

Una vez tengamos el Dockerfile completo podemos generar la imagen:

```docker
docker build -t vb6 .
```

E incluso ejecutarlo para comprobar que está todo funcionando:

```cmd
docker run -d -p 9001:9001 vb6
```

Una vez tengamos el contenedor en marcha podremos conectar al mismo utlizando el comando telnet de la siguiente forma: 

```cmd
telnet localhost 9001
```

Y deberíamos ver en el terminal el nombre de nuestro servidor, que en este caso se corresponderá con el identificador del contenedor:

![Telnet terminal with the text 'Connected to server: c433ff27eb8f'][telnet-connected]



## Sigue jugando

Ahora ya tenemos esa aplicación antigua hecha en VB6 dentro de nuestro ciclo de DevOps. Estamos generando una imagen de contenedor que es más fácil de mover y desplegar que una imagen de máquina virtual. En el [próximo capítulo][chapter-ii] de la serie veremos cómo podemos desplegar este contenedor en Kubernetes para poder aprovechar las ventajas que nos ofrece, como por ejemplo que nuestra aplicación vuelva a levantarse automáticamente en caso de fallo.

## Enlaces

* El [código en GitHub][ghcode]
* [Docker Desktop][docker-desktop]



[ghcode]: https://github.com/jmservera/legacyvb6ink8s
[docker-desktop]: https://www.docker.com/products/docker-desktop
[telnet-connected]: /008-using-k8s-to-run/telnet-connected.png "Telnet terminal"

[chapter-i]:{{< relref "#aplicación-de-ejemplo-vb6" >}}
[chapter-ii]:{{< ref "posts/008-using-k8s-to-run-legacy-vb6-ii.md" >}}
