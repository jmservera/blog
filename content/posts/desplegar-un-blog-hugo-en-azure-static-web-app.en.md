---
title: "How to deploy an HUGO site in Azure Static Web App"
images: [/desplegar-un-blog-hugo/createstaticwebapp.png]
date: 2020-05-21T23:24:26+02:00
draft: false
---

It's been over 5 years since I don't write on my personal blog, and since then I missed it a lot, but I've been focusing in other things  and I have lots of articles in my unfinished folder. But, on the 19th of this month, while I was watching the [Static Web App](https://mybuild.microsoft.com/sessions/898230c4-1350-4fc6-acba-6baf1a58d76a?source=sessions) presentation, I decided to test it with a  [HUGO](https://gohugo.io) blog. It's incredibly easy, but I wanted to share with you some small details so you find it even easier, since, remember, it's still a preview on May 21 2020.

> **Disclaimer**: there is an advanced tutorial with a custom HUGO build here: https://docs.microsoft.com/en-us/azure/static-web-apps/publish-hugo

## Blog creation

Debo confesar que soy un completo novato en esto de los blogs estáticos. Usé [Wordpress](https://jmservera.wordpress.com) durante muchos años, pero es verdad que 5 años más tarde todo ha cambiado y utilizo [Markdown](https://daringfireball.net/projects/markdown/) hasta para mis recetas de cocina, así que esto no puede ser tan difícil. Como este artículo no va de cómo crear un blog tenéis una guía fantástica en la web de [HUGO](https://gohugo.io/getting-started/quick-start/). Vamos a realizar los pasos básicos:

1. Para poder integrar nuestro sitio estático con la generación del blog, necesitamos que nuestro código este en GitHub, así que creamos un repositorio para nuestro blog:
  ![Creación del repositorio en GitHub][repo-create]

2. Luego creamos un blog con [hugo](https://gohugo.io), yo he instalado [HUGO 0.59.1](https://github.com/gohugoio/hugo/releases/tag/v0.59.1) que es el que por ahora es 100% compatible con las acciones de GitHub que se van a ejecutar luego. En la imagen veis los pásos básicos que he realizado:
  ![Usamos Hugo para crear un blog con un tema][hugo-create]
  Podéis ver que he hecho lo siguiente:
  1. He clonado el respositorio
  2. He creado un nuevo sitio con HUGO: ```hugo new site blog_es```
  3. He añadido el tema bootstrap-4 usando: ```git submodule add https://github.com/alanorth/hugo-theme-bootstrap4-blog themes/bootsrap4-blog```
  4. He aplicado el tema y he creado este post con ```hugo new posts/....```

## Despliegue de nuestro blog en Azure Static Web App

Una vez que ya tenemos nuestro blog preparado con HUGO y subido a github, podemos crear en Azure una nueva [Web App Estática](https://azure.microsoft.com/en-us/services/app-service/static/).

![Creación de la Static Web App][webapp-create]

Tendremos que configurar nuestras credenciales de GitHub:

![Configuración de la web][webapp-config]

Y también hay que cambiar la entrada App artifact location al valor *public* porque es la carpeta de salida por defecto de Hugo:

![Configurar dónde buscará la GitHub Action el contenido][webapp-config-artifact]

Esto nos creará una [GitHub Action](https://github.com/features/actions) que ejecutará automáticamente hugo para generar nuestro blog y desplegarlo en nuestra nueva web app, pero el primer despliegue no funcionará. Si recordáis, para crear añadir un tema en HUGO hemos utilizado un submódulo, así que tendremos que modificar el paso de *checkout* en la acción para que se descargue también los submódulos que contienen nuestro tema. En la carpeta *.github/workflows* encontraremos un archivo yml, ahí modificaremos el primer paso así: 

``` yaml
    - uses: actions/checkout@v2
      with:
        submodules: true
```
Y al hacer un push a nuestro repositorio se desplegará nuestro blog automáticamente.

![Blog picture][blog-picture]

[repo-create]: /desplegar-un-blog-hugo/createrepo.png "Crea un repositorio en GitHub"
[hugo-create]: /desplegar-un-blog-hugo/createhugofirstpost.png "Crea el primer post con hugo"

[webapp-create]: /desplegar-un-blog-hugo/createstaticwebapp.png "Crea una web app estática"

[webapp-config]: /desplegar-un-blog-hugo/createstaticwebapp_2.png "Configurar repositorio de GitHub"

[webapp-config-artifact]: /desplegar-un-blog-hugo/createstaticwebapp_3.png "Configurar carpeta public como output de hugo"

[blog-picture]: /desplegar-un-blog-hugo/blogpicture_en.png "Blog picture"
