# **Infrastructure As Code & ARM Templates**
## Creating an Azure Git repository
## Creating a resource group
## Creating a service connection
## Creating a release pipeline
#### azure-pipelines.yml
- Installing the **Azure Pipelines** extension for VS Code is a good idea for intellisense, but not required.
- We will be using an Azure Pipelines **.yml** file to define our pipeline. Clone the git repository from step 1 and create ```azure-pipelines.yml``` at the top level of the directory.
- You can find a reference to the **YAML Schema** as defined by Microsoft [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema)
- We want the pipeline to be triggered by the master branch to add:
~~~~
trigger:
    - master
~~~~
- In order to deploy and ARM template with the ```Azure resource group deployment``` task, we only need to run 1 task: ```AzureResourceGroupDeployment@2```. Like so: (NOTE: this azure-pipelines.yml can be condensed into a more implicit form without stages or jobs, however I like being explicit where possible).

~~~~
trigger:
  - master

stages:
  - stage: Deploy
    displayName: Deploy ARM template
    jobs:
      - job: Deploy
        displayName: Deploy
        pool:
          vmImage: 'windows-2019'
        steps:
          - task: AzureResourceGroupDeployment@2
            inputs:
              azureSubscription: ARMServiceConnection
              action: Create Or Update Resource Group
              deploymentMode: Complete
              resourceGroupName: Multi
              templateLocation: Linked artifact    
              csmFile: $(System.DefaultWorkingDirectory)/ARMTemplates/azuredeploy.json
~~~~
- The ```azureSubscription``` property should be set to the ```connectionName``` that was defined in previous steps when creating our ***service connection***.
- We set the ```templateLocation``` property to be ```Linked artifact``` so that we can use the ```csmFile``` property. ```csmFile``` will be used to specify the path to to the ARM template we created. 
- We set the ```csmFile``` to use the pre-defined variable ```$(System.DefaultWorkingDirectory)```, as it represents the local path on the agent where the source code files are downloaded (ie from **Azure Git repositories**).
- We set the deploymentMode property to complete to ensure our repository represents the current state of infrastructure currently deployed on Azure. See below for more details.
#### Deployment modes
When setting up a release pipeline to use the ```Azure resource group deployment``` task, you have to the option to specify the deployment mode.

- ***Incremental update (DEFAULT):*** Azure Resource Manager only makes changes to existing resources if they are specified in the ARM template. Pre-existing resources that are not referenced in the ARM template are left alone.  

- ***Complete update:*** Azure Resource Manager will delete all resources within a resource group that are not referenced in the ARM template. Essentially, the ARM template will reflect the current state of all the resources within a resource group.