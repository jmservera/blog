+++
title =  "Create and execute another pipeline inside an Azure DevOps pipeline itself"
date = 2021-03-05T11:00:16+01:00
featured_image= "https://live.staticflickr.com/4124/5057713273_e931c3a886_b.jpg"
tags = ["DevOps"]
draft = true
+++

If you need to create and/or execute pipelines from another Azure DevOps (ADO) pipeline, there's an ADO extension for the az CLI that will allow us to perform this task, but we will need some preparation steps before running it. Today it's still a preview characteristic, so it may suffer some changes when reaching general availability.

<!--more-->

## How to prepare our DevOps project

After the project and repository creation, we will need to provide some additional permissions to the service principal that will be used for the creation of new *pipelines*. If you don't want to mix the automatically created pipelines, we can create and assign the permissions "Edit build pipeline" and "Queue builds" to the build service account, which is usually created under the name "[Project name] Build Servcie (user alias)".

To do this, in the Pipelines section, under "All Pipelines" we create a folder (I created one called *automatic*). We open the menu and select the security option, there you can assign this permissions to the build service:

![Build and edit permissions to the Build Service account][folder-permissions]

## Project structure

In the repo, we will create a *pipelines* folder, with a *yaml* file for the pipeline. I called it *base-auto-pipeline.yml*. We write insed the pipeline definition we want to create and execute dynamically. For this example I will create a very simple one that will receive a variable, just for testing purposes:

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

Now that we have the pipeline definition that we want to create dinamically, we will create the actual pipeline that will do the job. We can use a "Starter pipeline" for this:

![Starter Pipeline creation][starter-pipeline]

And write this code inside it:

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

## Step by step explanation

In the first section, I just disabled the triggers so we will run the pipeline manually, I added two parameters to show them in the execution form and some variables we will use in the automation.

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

The two first steps in the pipeline are to prepare the environment. The Azure CLI installed in the ADO agent comes with the DevOps [DevOps](https://docs.microsoft.com/en-us/cli/azure/ext/azure-devops/pipelines?view=azure-cli-latest) extension that will help us interact with our pipelines. We don't need a Service Connection in this case because the credentials are already provided in a system variable, using the project name *$(System.TeamProject)* and the Personal Access Token (PAT) *$(System.AccessToken)* to perform the CLI login:

```yaml
steps:
- script: az devops configure --defaults organization='$(System.TeamFoundationCollectionUri)' project='$(System.TeamProject)' --use-git-aliases true
  displayName: 'Set default Azure DevOps organization and project'
  
- script: echo ${AZURE_DEVOPS_CLI_PAT} | az devops login
  env:
    AZURE_DEVOPS_CLI_PAT: $(System.AccessToken)
  displayName: 'Login Azure DevOps Extension'
```

In the next step, we check if the pipeline with the same name already exists, and then we use the [az pipelines create](https://docs.microsoft.com/en-us/cli/azure/ext/azure-devops/pipelines?view=azure-cli-latest#ext_azure_devops_az_pipelines_create) command to create one with the provided name. We are using the previously created parameter ``` ${{ parameters.automationName }} ```, using the macro language because these values are not passed as environment variables in this case.

We are also using system variables, as the repo name, branch, etc., we provide the path to the yaml file that will be used to create the pipeline and, finally, we store the pipeline identifier that we wil use afterwards.

We also use the *--skip-first-run* argument because we want to execute this pipeline in a next step to pass some additional values. This could be useful if we want to add a manual approval or split the work in different stages.

> Note: I **didn't use** the [Jobs and Stages][pipeline-phases] separation, because in each job the context is restarted, this would force us to perform the ADO login in each phase, and by now this step takes a long time and would slower our pipeline. Take this in mind if you want to split the pipeline in different jobs or stages.

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

One interesting characteristic is that we are using the same pipeline definition file (the pipelines/base-auto-pipeline.yml) to create many pipelines from it, so, modifying this one yaml file would update all the pipelines created from it.

There's a special line used to fill an environment variable that will be needed in subsequent steps. Each *script* or *bash* step is executed inside a new .sh file with the provided script, so, to guarantee we can pass values through the different contexts we need to rely on the ADO pipelines agent to do it by injecting the variables:

```bash
    echo "##vso[task.setvariable variable=pipelineId]$pipelineId"
```
> Note: if you are going to use stages, you need [to point out that you are assigning an *output*][output-variable] to pass the information between the different jobs.

Finally, we execute the pipeline with the id we obtained before and will use the parameter to have a conditional execution. We can assign some values to the pipeline using the command line, like we can see here with the *testVariable*:

```yaml
- ${{ if eq(parameters.runThePipeline, true) }}:
  - script: |
      echo "run $(pipelineId)"
      az pipelines run --id $(pipelineId) --branch '$(Build.SourceBranchName)' --variables testVariable='Hello world!'
    displayName: "Run the pipeline"
```


## Pipeline execution

Now , we can execute the pipeline. In the execution form we will find the different parameters we indicated before:

![pipelinerun][run-pipeline]

## Link summary

* [Variable definition in ADO][define-variables] 
* [Pipeline creation using the CLI][az-pipelines-create]
* [Typed parameters in ADO pipelines][runtime-parameters]
* [Pipeline phases][pipeline-phases]
* [Outputs between jobs][output-variable]

## Credits

* Header picture by [Pierre-Alexandre Garneau][pierre-inception].

[az-pipelines-create]: https://docs.microsoft.com/en-us/cli/azure/ext/azure-devops/pipelines?view=azure-cli-latest#ext_azure_devops_az_pipelines_create
[define-variables]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch
[pipeline-phases]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/phases?view=azure-devops&tabs=yaml
[output-variable]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#set-a-multi-job-output-variable
[runtime-parameters]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/runtime-parameters?view=azure-devops&tabs=script

[folder-permissions]: /006-crear-y-ejecutar-pipelines-desde/folder%20permissions.png "Pipelines' folder permission"
[starter-pipeline]: /006-crear-y-ejecutar-pipelines-desde/starter_pipeline.png "New pipeline"
[ado-variable]: /006-crear-y-ejecutar-pipelines-desde/ado_variable.png "ADO Variable"
[run-pipeline]: /006-crear-y-ejecutar-pipelines-desde/run_pipeline.png

[pierre-inception]: https://www.flickr.com/photos/pagarneau/ "Inception"