---
title: "How to deploy an HUGO site in Azure Static Web App"
featured_image: "/images/1500x500.jfif"
date: 2020-05-21T23:24:26+02:00
draft: false
---

The past 19th of this month, while I was watching the [Static Web App](https://mybuild.microsoft.com/sessions/898230c4-1350-4fc6-acba-6baf1a58d76a?source=sessions) Build presentation, I decided to do a test with [HUGO](https://gohugo.io) to start again my blog after 5 years of silence.

TL;DR: [Static Web Apps](https://docs.microsoft.com/azure/static-web-apps/) allows you to deploy static content with a custom domain and Azure Web Apps does the rest: creates a GitHub Action for continuous deployment, gives you a SSL certificate for your custom domain, does the global distribution and helps you with Azure Functions if you need to generate some content from an API. Best of all, all for FREE, even the certificate for your site.

<!--more-->

> **Disclaimer**: I did this before reading the [official tutorial](https://docs.microsoft.com/azure/static-web-apps/publish-hugo) where you will find an advanced step-by-step guide for a customized HUGO deployment.

## Blog creation

I must confess, I'm a total rookie with static blogs. I used [Wordpress](https://jmservera.wordpress.com) during so many years, but this last 5 years everything changed and I'm using [Markdown](https://daringfireball.net/projects/markdown/) even for my cooking recipes, so this shouldn't be too hard. This article is not about blogging, so you have an awesome guide on how to tart in the [HUGO](https://gohugo.io/getting-started/quick-start/) website. Let's summarize the basic steps:

1. First we will need a GitHub repo to integrate it with Azure:
  ![Creación del repositorio en GitHub][repo-create]

2. Then we create a simple blog with the [hugo](https://gohugo.io) CLI. See in the picture the basic steps:
  ![Usamos Hugo para crear un blog con un tema][hugo-create]

  What I've done is:

  1. I cloned the repo
  2. I created a HUGO new site:
    ```
    hugo new site blog_es --force
    ```
  3. I added a them as a submodule:
    ```
    git submodule add https://github.com/alanorth/hugo-theme-bootstrap4-blog themes/bootsrap4-blog
    ```
  4. And after applying the theme I created a new post
    ```
    hugo new posts/desplegar-un-blog-hugo-en-azure-static-web-app
    ```

## Deploy our new blog in Azure Static Web App

Once we have our new blog prepared and pushed to GitHub we create in Azure a new [Static Web App](https://azure.microsoft.com/services/app-service/static/).

![Creación de la Static Web App][webapp-create]

We will need to configure our new GitHub credentials:

![Configuración de la web][webapp-config]

We will need to change the App artifact location to *public*, the folder where HUGO creates the output:

![Configurar dónde buscará la GitHub Action el contenido][webapp-config-artifact]

And this will create for us a [GitHub Action](https://github.com/features/actions) that will generate and deploy our blog in our web app, but the first deployment will fail. You will remember that I used a submodule for the HUGO theme, so, we will need to modify the *checkout* action to force the download of the submodule. in the *.github/workflows* folder there's a *.yml* file that we will need to modify in the checkout action: 

``` yaml
    - uses: actions/checkout@v2
      with:
        submodules: true
```

After the push the blog will finally deploy into our static web app:

![Blog picture][blog-picture]

## How all this works

The magic behind this is supported by two different systems:

* Azure provides all the infrastructure from the already existing App Services, like Custom domains, deployment slots (called Environments), authentication and role authorization, routes and Azure Functions integration for a stateless API.
* GitHub has the GitHub Actions system that takes care of the static web content generation from your repo, and then do the automated deployment to the static web app. For this task, you can take a look to the [Azure/static-web-apps-deploy](https://github.com/Azure/static-web-apps-deploy) repo where you will see how a tool named Oryx is used. This tool detects the language and builds your repo. The supported platforms are: dotnet, nodejs, php and python, but can also generate static web apps from HUGO 0.59 or multilanguage sites like Django + React.

If the build and deploy script does not find something to build, it will deploy whatever is found in the main folder, with the condition that an index page is present in the folder. That's why the official example adds some extra steps to use a newer version of HUGO:

``` yaml
    - name: Setup Hugo
      uses: peaceiris/actions-hugo@v2.4.8
      with:
        hugo-version: "latest"
    - name: Build
      run: hugo
```

And thats all folks. Now I have to search for a good [theme template](https://themes.gohugo.io/) for my blog.

[repo-create]: /desplegar-un-blog-hugo/createrepo.png "Crea un repositorio en GitHub"
[hugo-create]: /desplegar-un-blog-hugo/createhugofirstpost.png "Crea el primer post con hugo"

[webapp-create]: /desplegar-un-blog-hugo/createstaticwebapp.png "Crea una web app estática"

[webapp-config]: /desplegar-un-blog-hugo/createstaticwebapp_2.png "Configurar repositorio de GitHub"

[webapp-config-artifact]: /desplegar-un-blog-hugo/createstaticwebapp_3.png "Configurar carpeta public como output de hugo"

[blog-picture]: /desplegar-un-blog-hugo/blogpicture_en.png "Blog picture"
