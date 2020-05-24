---
title: "How to deploy a HUGO site in Azure Static Web App"
featured_image: "/images/1500x500.jfif"
date: 2020-05-21T23:24:26+02:00
draft: false
---

This past May 19th, while I was watching the [Static Web App](https://mybuild.microsoft.com/sessions/898230c4-1350-4fc6-acba-6baf1a58d76a?source=sessions) Build presentation, I decided to do a test with [HUGO](https://gohugo.io) to start again my blog after 5 years of silence.

TL;DR: [Static Web Apps](https://docs.microsoft.com/azure/static-web-apps/) allows you to deploy static content with a custom domain and Azure Web Apps does the rest: creates a GitHub Action for continuous deployment, gives you an SSL certificate for your custom domain, does the global distribution and helps you with Azure Functions if you need to generate some content from an API. Best of all, it's all for FREE, you even get a free SSL certificate for your site's custom domain.

<!--more-->

> **Disclaimer**: I did this before reading the [official tutorial](https://docs.microsoft.com/azure/static-web-apps/publish-hugo) where you will find an advanced step-by-step guide for a customized HUGO deployment.

## First, you need a blog

I must confess I'm a total noobie with static blogs. I used [WordPress](https://jmservera.wordpress.com) for so many years, but this last 5 years everything changed and I'm using [Markdown](https://daringfireball.net/projects/markdown/) even for my cooking recipes, so this shouldn't be too hard. This article is not about blogging, and you have an awesome guide on how to start on the [HUGO](https://gohugo.io/getting-started/quick-start/) website. But, let's summarize the basic steps:

1. First, we will need a Git repo, this is how HUGO works and Azure Static Web Apps uses GitHub to link your repo, so here we go:
  ![GitHub repository creation][repo-create]

2. Then we create a simple blog with the [HUGO](https://gohugo.io) CLI. See in this picture my basic steps:

   ![Use HUGO to create a blog][HUGO-create]

   What I've done is:

   1) I cloned the *almost* empty repo
   1) Then I created a new site with HUGO:

        ```bash
        hugo new site blog_es --force
        ```

   1) The theme is added as a submodule:

      ```bash
      git submodule add https://github.com/alanorth/hugo-theme-bootstrap4-blog themes/bootsrap4-blog
      ```

   1) And, finally, I created this post:

      ```bash
      hugo new posts/desplegar-un-blog-hugo-en-azure-static-web-app
      ```

After these steps we have a plain and simple blog that we can edit and test locally.

## Deploy our new blog in Azure Static Web App

Once we have pushed the blog to GitHub, we create in Azure a new [Static Web App](https://azure.microsoft.com/services/app-service/static/).

![Static Web App Creation][WEBAPP-create]

We will need to configure our GitHub credentials:

![Web app configuration][WEBAPP-config]

And change the App artifact location to "*public*", the folder where HUGO creates the output:

![Change where the GitHub Action will search for content][WEBAPP-config-artifact]

And this will automagically create a [GitHub Action](https://github.com/features/actions) that will generate and deploy our blog in our web app. The first deployment will fail because I used a submodule for the HUGO theme and this is missing by now, so, we will need to modify the *checkout* action to force the download of the submodule. In the *.github/workflows* folder there's a *.yml* file that we will need to modify in the checkout step:

```yml
- uses: actions/checkout@v2
  with:
    submodules: true
```

Pushing these changes will run the action again that will now deploy our blog into our static web app:

![Blog picture][blog-picture]

## How all this works

The magic behind this is supported by two different systems:

* **Azure** provides all the infrastructure. As my colleague [@CJ_Aliaga](https://twitter.com/CJ_Aliaga) told me, all this systems already existed before: Custom domains integration, SSL, deployment slots (called Environments), authentication and role authorization, routes, and Azure Functions integration for a stateless API.
* **GitHub Actions** system that takes care of the static web content generation from your repo, and then do the automated deployment to the static web app. You can see how it's done in the [Azure/static-web-apps-deploy](https://github.com/Azure/static-web-apps-deploy) repo, where you will find how a tool named [Oryx](https://github.com/microsoft/Oryx) is used. This tool detects the language and builds your repo when it is created in one of the supported platforms: .Net, Nodejs, PHP, and Python. It can also generate static web apps from HUGO 0.59, and also supports sites using a combination of programming languages like, for example, Django + React.

If the build and deploy script does not find something to build, but it finds an index page in the main folder, it will deploy whatever is found there. That is why the official example adds some extra steps to use a newer version of HUGO:

``` yaml
- name: Setup Hugo
  uses: peaceiris/actions-hugo@v2.4.8
  with:
    hugo-version: "latest"
- name: Build
  run: hugo
```

This means that **you can use any static site generator** that can be run from the command line.

And that's all folks. Now I have to search for a good [theme template](https://themes.gohugo.io/) for my blog. See you around.

[repo-create]: /desplegar-un-blog-hugo/createrepo.png "GitHub repository creation"
[HUGO-create]: /desplegar-un-blog-hugo/createhugofirstpost.png "Create the first post with HUGO"

[WEBAPP-create]: /desplegar-un-blog-hugo/createstaticwebapp.png "Crete a Static Web App"

[WEBAPP-config]: /desplegar-un-blog-hugo/createstaticwebapp_2.png "Configure the GitHub repository link"

[WEBAPP-config-artifact]: /desplegar-un-blog-hugo/createstaticwebapp_3.png "Configurate the public folder as the output from HUGO"

[blog-picture]: /desplegar-un-blog-hugo/blogpicture_en.png "Picture of this blog post"
