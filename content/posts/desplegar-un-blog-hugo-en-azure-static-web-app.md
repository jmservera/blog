---
title: "Desplegar Un Blog Hugo en Azure Static Web App"
date: 2020-05-21T23:24:26+02:00
draft: true
---

## Creación del blog

Creamos un repo en github

![alt text][repo-create]

Luego creamos un blog con [hugo](https://gohugo.io):

![alt text][hugo-create]

Creamos en Azure una [Web App Estática](https://azure.microsoft.com/en-us/services/app-service/static/)

![alt text][webapp-create]

Tendremos que configurar nuestras credenciales de GitHub

![alt text][webapp-config]

También hay que cambiar la entrada App artifact location al valor *public* porque es la carpeta de salida por defecto de Hugo:

![alt text][webapp-config-artifact]


Esto nos creará una GitHub Action que ejecutará automáticamente hugo para generar nuestro blog y desplegarlo en nuestra nueva web app, pero en el caso de hugo vamos a necesitar modificar el paso de checkout para que se descargue también los submódulos que contienen nuestro tema. En la carpeta .github/workflows encontraremos un archivo yml, ahí modificaremos el primer paso así: 

``` yaml
    - uses: actions/checkout@v2
      with:
        submodules: true
```
Y al hacer un push a nuestro repositorio se desplegará nuestro blog automáticamente.


[repo-create]: /static/desplegar-un-blog-hugo/createrepo.png "Crea un repositorio en GitHub"
[hugo-create]: /static/desplegar-un-blog-hugo/createhugofirstpost.png "Crea el primer post con hugo"

[webapp-create]: /static/desplegar-un-blog-hugo/createstaticwebapp.png "Crea una web app estática"

[webapp-config]: /static/desplegar-un-blog-hugo/createstaticwebapp_2.png "Configurar repositorio de GitHub"

[webapp-config-artifact]: /static/desplegar-un-blog-hugo/createstaticwebapp_3.png "Configurar carpeta public como output de hugo"