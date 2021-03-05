+++
title =  "Crear y ejecutar pipelines desde otro pipeline en Azure DevOps"
date = 2021-03-05T11:00:16+01:00
featured_image= "https://live.staticflickr.com/4124/5057713273_e931c3a886_b.jpg"
tags = ["DevOps"]
draft = false
+++

Si os habéis encontrado con la necesidad de crear y ejecutar pipelines de Azure DevOps (ADO) desde una pipeline, existe una extensión de devops para el az CLI que nos permitirá hacer esto, aunque tendremos que preparar un poco el proyecto de ADO antes. A día de hoy todavía es una característica en preview, así que puede que algo de lo que describa aquí sufra cambios antes de estar en disponibilidad general.

<!--more-->

## Preparar nuestro proyecto de Azure DevOps

Tras crear un proyecto con un repositorio, vamos a necesitar dar permisos al service principal que se usa para crear nuevas *pipelines* en algún sitio. Para no mezclarlas con las pipelines que hayamos creado manualmente, podemos crear una carpeta y asignar los permisos "Edit build pipeline" y "Queue builds" a la cuenta del servicio de build, que suele tener el nombre "[Nombre del proyecto] Build Service (alias de usuario)". 
Para hacer esto, vamos al apartado de Pipelines y en el apartado "All Pipelines" creamos una carpeta. A la derecha de la carpeta pulsamos sobre el menú y abrimos la opción security, ahí podemos asignar los permisos al usuario del servicio de build:

![Asignación de permisos de edición y encolado a la cuenta de servicio][folder-permissions]

## Creación de la estructura

En el repositorio de código, vamos a añadir una carpeta llamada *pipelines* y dentro un archivo yaml, que se llame, por ejemplo, *base-auto-pipeline.yml*, con la definición de la pipeline que queremos crear y ejecutar dinámicamente. Para el ejemplo usaré una muy sencilla que recibirá una variable llamada *testVariable*:

```yaml
trigger: none

pool:
  vmImage: ubuntu-latest

steps:

- script: |
    echo Congrats, your variable was set to:
    echo $(testVariable)
  displayName: 'Run a test script'
```

Luego vamos a crear una nueva pipeline a través de la sección de pipelines, usaremos una "Starter pipeline":

![Creación de una nueva Starter Pipeline][starter-pipeline]

Y la modificaremos inmediatamente con un yaml como este:

```yaml
# manual trigger only
trigger: none

parameters:
  # this parameter is required, no default value
  - name: automationName
    type: string
  # used to conditionally run the last step
  - name: runThePipeline
    type: boolean
    default: true
    displayName: Run the generated pipeline

pool:
  vmImage: ubuntu-latest

variables:
  - name: automationFolder
    value: automatic
  - name: sourcePipeline
    value: pipelines/base-auto-pipeline.yml
  - name: pipelineId

steps:

- script: az devops configure --defaults organization='$(System.TeamFoundationCollectionUri)' project='$(System.TeamProject)' --use-git-aliases true
  displayName: 'Set default Azure DevOps organization and project'
  
- script: echo ${AZURE_DEVOPS_CLI_PAT} | az devops login
  env:
    AZURE_DEVOPS_CLI_PAT: $(System.AccessToken)
  displayName: 'Login Azure DevOps Extension'

- script: |
    pipelineId=$(az pipelines show --folder-path $(automationFolder) --name '${{ parameters.automationName }}' --query id -o tsv)
    echo $pipelineId
    if [ $pipelineId ]
    then
      echo "Pipeline already exists with id $(pipelineId)"
    else
      pipelineId=$(az pipelines create --name '${{ parameters.automationName }}' --folder-path $(automationFolder) --branch '$(Build.SourceBranchName)' --repository '$(Build.Repository.Name)' --repository-type $(Build.Repository.Provider) --yaml-path '$(sourcePipeline)' --skip-first-run true --query id -o tsv)
    fi

    echo "pipeline id $pipelineId"
    echo "##vso[task.setvariable variable=pipelineId]$pipelineId"
  displayName: "Create pipeline if not exists"

- ${{ if eq(parameters.runThePipeline, true) }}:
  - script: |
      echo "run $(pipelineId)"
      az pipelines run --id $(pipelineId) --branch '$(Build.SourceBranchName)' --variables testVariable='Hello world!'
    displayName: "Run the pipeline"
```

## Explicación de cada paso de la pipeline

En la primera sección, he deshabilitado el trigger, he añadido dos parametros para que nos aparezcan en el formulario de ejecución, y algunas variables que usaremos dentro de la automatización.

```yaml
# manual trigger only
trigger: none

parameters:
  # this parameter is required, no default value
  - name: automationName
    displayName: Automation name
    type: string
  # used to conditionally run the last step
  - name: runThePipeline
    type: boolean
    default: true
    displayName: Run the generated pipeline

pool:
  vmImage: ubuntu-latest

variables:
  - name: automationFolder
    value: automatic
  - name: sourcePipeline
    value: pipelines/base-auto-pipeline.yml
  - name: pipelineId
```

Los dos primeros pasos del pipeline son para preparar el entorno. El Azure CLI que viene instalado en el agente de Azure DevOps ya tiene instalada la extensión de [DevOps](https://docs.microsoft.com/en-us/cli/azure/ext/azure-devops/pipelines?view=azure-cli-latest) que nos permitirá interactuar con nuestros pipelines. Este caso es un poco diferente del que nos encontramos cuando tenemos que conectar a Azure, donde usamos una Service Connection; aquí tenemos que configurarlo mediante las variables de sistema que nos proporcionan, por ejemplo, el nombre del proyecto *$(System.TeamProject)* o el Personal Access Token (PAT) *$(System.AccessToken)* para realizar el login en el CLI:

```yaml
steps:
- script: az devops configure --defaults organization='$(System.TeamFoundationCollectionUri)' project='$(System.TeamProject)' --use-git-aliases true
  displayName: 'Set default Azure DevOps organization and project'
  
- script: echo ${AZURE_DEVOPS_CLI_PAT} | az devops login
  env:
    AZURE_DEVOPS_CLI_PAT: $(System.AccessToken)
  displayName: 'Login Azure DevOps Extension'
```

En el siguiente paso, nos encontramos con un script que mira si la pipeline que queremos crear ya existe con ese nombre, para no volver a crearla. En caso negativo, usa el comando [az pipelines create](https://docs.microsoft.com/en-us/cli/azure/ext/azure-devops/pipelines?view=azure-cli-latest#ext_azure_devops_az_pipelines_create) para crear una nueva con el nombre que le queramos dar. Fijaos que aquí usamos el parámetro ``` ${{ parameters.automationName }} ```, que se incrusta usando el lenguaje de plantillas propio de pipelines, aquí este valor no entra como variable de entorno como en otros casos.
También estamos usando variables de sistema como el nombre del repositorio, el nombre de la rama, etc. Además proporcionamos un path al archivo yaml que se usará para generar la pipeline y, finalmente, nos guardamos el identificador de la pipeline para poderlo usar luego.

Usamos el parámetro *--skip-first-run* porque queremos ejecutar esa pipeline en un siguiente paso y pasarle alguna variable adicional, esto, además, nos permitiría meter ese paso dentro de un *stage* de forma que podamos introducir un proceso de aprobación manual o cualquier otra comprobación que necesitemos realizar antes de ejecutar la siguiente pipeline.

> Nota: **no** he utilizado la separación de [fases en Jobs y Stages][pipeline-phases], porque en cada job se reincia el contexto, eso nos obligaría a tener que volver a realizar el login en ADO y ahora mismo ese proceso es bastante lento. Si necesitáis separar en diferentes etapas, recordad que hay que volver a ejecutar el proceso de login.

```bash
pipelineId=$(az pipelines show --folder-path $(automationFolder) --name '${{ parameters.automationName }}' --query id -o tsv)
echo $pipelineId
if [ $pipelineId ]
then
    echo "Pipeline already exists with id $(pipelineId)"
else
    pipelineId=$(az pipelines create --name '${{ parameters.automationName }}' --folder-path $(automationFolder) --branch '$(Build.SourceBranchName)' --repository '$(Build.Repository.Name)' --repository-type $(Build.Repository.Provider) --yaml-path '$(sourcePipeline)' --skip-first-run true --query id -o tsv)
fi
```

Una característica interesante es que estamos utilizando el mismo archivo de definición de pipeline (el pipelines/base-auto-pipeline.yml) para crear tantas pipelines como queramos, así que si modificamos el proceso en el yaml, todas las pipelines que dependan del mismo estarán actualizadas.

Hay una línea más un tanto particular; se utiliza para rellenar la variable de entorno que se pasará a los siguientes pasos. Cada entrada *script* o *bash* ejecuta un .sh nuevo con el contenido que le estemos pasando, así que cualquier variable de entorno que asignemos ahí no pasará al siguiente script y necesitamos que el agente se encargue de inyectar esas variables.

```bash
    echo "##vso[task.setvariable variable=pipelineId]$pipelineId"
```
> Nota: si luego empezáis a utilizar etapas, hay que [indicar que estamos asignando un *output*][output-variable] para pasar información entre diferentes jobs.

Para acabar, ejecutamos la pipeline con el id que hemos obtenido en el paso anterior y usamos el parámetro para ejecutar ese paso de forma condicional. A la pipeline le podemos asignar variables a través de esa línea de comando:

```yaml
- ${{ if eq(parameters.runThePipeline, true) }}:
  - script: |
      echo "run $(pipelineId)"
      az pipelines run --id $(pipelineId) --branch '$(Build.SourceBranchName)' --variables testVariable='Hello world!'
    displayName: "Run the pipeline"
```


## Ejecución de la pipeline

Y ahora ya podemos ejecutar la pipeline, en el formulario de ejecución nos aparecerán los diferentes parámetros que hayamos indicado:

![pipelinerun][run-pipeline]

## Resumen de enlaces

* [Definición de variables][define-variables] 
* [Creación de pipelines desde el CLI][az-pipelines-create]
* [Uso de parámetros tipificados en pipelines][runtime-parameters]
* [Fases en pipelines][pipeline-phases]
* [Outputs entre diferentes jobs][output-variable]

## Créditos

* Foto de portada por [Pierre-Alexandre Garneau][pierre-inception].

[az-pipelines-create]: https://docs.microsoft.com/en-us/cli/azure/ext/azure-devops/pipelines?view=azure-cli-latest#ext_azure_devops_az_pipelines_create
[define-variables]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch
[pipeline-phases]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/phases?view=azure-devops&tabs=yaml
[output-variable]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#set-a-multi-job-output-variable
[runtime-parameters]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/runtime-parameters?view=azure-devops&tabs=script

[folder-permissions]: /006-crear-y-ejecutar-pipelines-desde/folder%20permissions.png "Permisos sobre la carpeta de pipelines"
[starter-pipeline]: /006-crear-y-ejecutar-pipelines-desde/starter_pipeline.png "Nueva pipeline"
[ado-variable]: /006-crear-y-ejecutar-pipelines-desde/ado_variable.png "Variable ADO"
[run-pipeline]: /006-crear-y-ejecutar-pipelines-desde/run_pipeline.png

[pierre-inception]: https://www.flickr.com/photos/pagarneau/ "Inception"