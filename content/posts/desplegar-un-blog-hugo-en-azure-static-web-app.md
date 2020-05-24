---
title: "Desplegar Un Blog Hugo en Azure Static Web App"
featured_image: "/images/1500x500.jfif"
date: 2020-05-21T23:24:26+02:00
draft: false
---

El pasado día 19, mientras estaba mirando la presentación de las [Static Web App](https://mybuild.microsoft.com/sessions/898230c4-1350-4fc6-acba-6baf1a58d76a?source=sessions) en el Build, decidí hacer una prueba con [HUGO](https://gohugo.io) para retomar mi blog tras 5 años de inactividad, 

TL;DR: [Static Web Apps](https://docs.microsoft.com/azure/static-web-apps/) te permite desplegar contenido estático y asignarle un dominio, Azure Web Apps se encarga del resto: creación de una GitHub Action para despliegue contínuo, SSL para nuestro dominio personalizado, distribución global y llamar a alguna Azure Function si hace falta alguna pequeña parte dinámica. Y lo mejor de todo es que es GRATIS, sí, incluso el certificado SSL para tu sitio.

<!--more-->

> **Aviso**: todo esto lo hice sin mirar antes la [explicación oficial](https://docs.microsoft.com/azure/static-web-apps/publish-hugo), donde podréis encontrar una guía avanzada con un despliegue personalizado de HUGO.


## Creación del blog

Debo confesar que soy un completo novato en esto de los blogs estáticos. Usé [Wordpress](https://jmservera.wordpress.com) durante muchos años, pero es verdad que 5 años más tarde todo ha cambiado y utilizo [Markdown](https://daringfireball.net/projects/markdown/) hasta para mis recetas de cocina, así que esto no puede ser tan difícil. Como este artículo no va de cómo crear un blog tenéis una guía fantástica en la web de [HUGO](https://gohugo.io/getting-started/quick-start/).

Resumo los pasos básicos:

1. Para poder integrar nuestro sitio estático con la generación del blog, necesitamos que nuestro código este en GitHub, así que creamos un repositorio para nuestro blog:
  ![Creación del repositorio en GitHub][repo-create]

2. Luego creamos un blog con el CLI de [hugo](https://gohugo.io). En la imagen veis los pásos básicos que he realizado:
  ![Usamos Hugo para crear un blog con un tema][hugo-create]
  Podéis ver que he hecho lo siguiente:
  1. He clonado el repositorio
  2. He creado un nuevo sitio con HUGO: 
  ``` bash
    hugo new site blog_es
  ```
  3. He añadido el tema bootstrap-4 usando: 
  ``` bash
    git submodule add https://github.com/alanorth/hugo-theme-bootstrap4-blog themes/bootsrap4-blog 
  ```
  4. He aplicado el tema y he creado este post con:
  ``` bash
    hugo new posts/desplegar-un-blog-hugo-en-azure-static-web-app
  ```

## Despliegue de nuestro blog en Azure Static Web App

Una vez que ya tenemos nuestro blog preparado con HUGO y subido a GitHub, podemos crear en Azure una nueva [Web App Estática](https://azure.microsoft.com/en-us/services/app-service/static/).

![Creación de la Static Web App][webapp-create]

Tendremos que configurar nuestras credenciales de GitHub:

![Configuración de la web][webapp-config]

Y también hay que cambiar la entrada App artifact location al valor *public* porque es la carpeta de salida por defecto de Hugo:

![Configurar dónde buscará la GitHub Action el contenido][webapp-config-artifact]

Esto nos creará una [GitHub Action](https://github.com/features/actions) que ejecutará automáticamente HUGO para generar nuestro blog y desplegarlo en nuestra nueva web app, pero el primer despliegue no funcionará. Si recordáis, para añadir un tema en HUGO hemos utilizado un submódulo, así que tendremos que modificar el paso de *checkout* en la acción para que se descargue también los submódulos que contienen nuestro tema. En la carpeta *.github/workflows* encontraremos un archivo yml, ahí modificaremos el primer paso así: 

``` yaml
    - uses: actions/checkout@v2
      with:
        submodules: true
```

Y al hacer un push a nuestro repositorio se desplegará nuestro blog automáticamente.

![Imagen del blog][blog-picture]

## Cómo funciona todo esto

La magia se realiza desde dos elementos completamente diferenciados:

* Desde Azure se proporciona toda la infraestructura que ya existía en Azure para App Services como son los Custom domains, Slots de despliegue (aquí se llaman Environments), autenticación y acceso por roles, control de rutas y, para poder tener una API stateless que sirva contenido dinámico, se integra con Azure Functions.
* En GitHub tenemos las GitHub Actions que realizan la tarea de generar la web estática en base al repositorio que hayamos proporcionado y desplegarla automáticamente a nuestro sitio estático. Para esta tarea en concreto podemos encontrar la acción [Azure/static-web-apps-deploy](https://github.com/Azure/static-web-apps-deploy) que utiliza una utilidad llamada [Oryx](https://github.com/microsoft/Oryx) que permite detectar automáticamente el lenguaje y compilar un repositorio. Si os fijáis en las plataformas que soporta son: .NET, Nodejs, PHP y Python. Pero también es capaz de generar sitios estáticos a partir de HUGO (usando 0.59) e incluso sitios creados en múltiples lenguajes como Django+React.

El script de la acción de compilación y despliegue, en el caso de que no encuentre código a compilar simplemente desplegará lo que encuentre en la carpeta que le hayamos indicado, por eso en el [ejemplo oficial](https://docs.microsoft.com/es-es/azure/static-web-apps/publish-hugo) modifican los pasos de la acción para generar el sitio y así poder utilizar la última versión en lugar de la que tiene Oryx internamente:

``` yaml
    - name: Setup Hugo
      uses: peaceiris/actions-hugo@v2.4.8
      with:
        hugo-version: "latest"
    - name: Build
      run: hugo
```

Y esto es todo por hoy, ahora me queda encontrar una [buena plantilla](https://themes.gohugo.io/) para el blog.
Saludos confinados!

[repo-create]: /desplegar-un-blog-hugo/createrepo.png "Crea un repositorio en GitHub"
[hugo-create]: /desplegar-un-blog-hugo/createhugofirstpost.png "Crea el primer post con hugo"

[webapp-create]: /desplegar-un-blog-hugo/createstaticwebapp.png "Crea una web app estática"

[webapp-config]: /desplegar-un-blog-hugo/createstaticwebapp_2.png "Configurar repositorio de GitHub"

[webapp-config-artifact]: /desplegar-un-blog-hugo/createstaticwebapp_3.png "Configurar carpeta public como output de hugo"

[blog-picture]: /desplegar-un-blog-hugo/blogpicture.png "Imagen del blog"
