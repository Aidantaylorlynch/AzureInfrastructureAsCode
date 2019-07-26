# **Infrastructure As Code & ARM Templates**
## Creating an Azure Git repository
#### Creating an additional Git repository within a project
- When you create a new project a Git repository is automatically created for you. I like to have a seperate repository to manage infrastructure so that the infrastructure history does not get mixed with your application history.
- To create a new Git repository navigate to your project on Azure Devops, then go to: ***Repos***.
- On the top of the page, click on the Git icon drop down menu and select ***New repository***.  


## Creating a resource group
#### Install the Azure CLI
- To create a new resource group we will use the Azure CLI, but this can be done using the **Azure Portal**. 
- Install on [Windows](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest)
- Install on [MacOS](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-macos?view=azure-cli-latest)  
#### Login and Create
- Open a terminal and use the login command. This will open up a browser and let you sign into your Microsoft account.
- Once signed in, it will list the Azure subscriptions that your account has access to.
~~~~
az login
~~~~
- Now that we are logged in we want to create the resource group using the create command:
~~~~
az group create -n myNewResourceGroupName --location australiasoutheast
~~~~
- The ```-n``` flag is required, and is the name of the resource group you want to create.
- The ```--location``` flag is also required, and specifies the location you want your resource group be to in.
- To get a list of available locations you can run the following command:
~~~~
az account list-locations
~~~~


## Creating a service connection
#### Resources in a pipeline
- As we are about to create a pipeline which needs access to the Azure Resource Manager [resource](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/resources?view=azure-devops), we need to create a new service connection.
- Navigate to your project in Azure Devops, then go to ***Project settings*** > ***Service connections*** > ***New service connection*** > ***Azure Resource Manager***
- Give the service connection a name: ```ARMServiceConnection```.
- Click OK. This service connection can now be used in our pipeline.  

***NOTE:*** Service connections are also needed whenever you add external [resources](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/resources?view=azure-devops) to your pipeline after the ***first deployment***. For example adding a new ```vmImage``` to the pipeline would require creating a new service connection **or** explicitly authorizing the resource using the classic Azure editor within the pipelines section of Azure Devops.


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
- In order to deploy and ARM template with the ```Azure resource group deployment``` task, we only need to run 1 task: ```AzureResourceGroupDeployment@2```. Like so: (***NOTE:*** this azure-pipelines.yml can be condensed into a more implicit form without stages or jobs, however I like being explicit where possible).

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

## Create an Azure Resource Manager Template (ARM Template)
Now that we have a pipeline definition, we need to create an ARM template to be able to provision some infrastructure. We will be deploying a storage account, but there are Lots of templates available for random scenarios [here](https://azure.microsoft.com/en-au/resources/templates/).

#### azuredeploy.json
- First create a new directory ```ARMTemplates``` that will store the ARM template.
- Create a new file called azuredeploy.json.
- Add the basic [structure](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authoring-templates) of a template:
~~~~
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "",
  "resources": [  ],
}
~~~~
- To see how to provision a new storage account with this template, open up a link to the [Microsoft reference material](https://docs.microsoft.com/en-au/azure/templates/).
- Using the tree list on the left-side of the page, navigate to ***Storage*** > ***{latest-date}*** > ***Storage Accounts*** > ***Storage Account***.
- Copy over the properties that are required/any others you want to use:
~~~~
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "",
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "name": "yoloswagginsstorage",
      "location": "australiasoutheast",
      "apiVersion": "2019-04-01",
      "sku": {
        "name": "Standard_LRS"
      },
      "kind": "StorageV2",
      "properties": {}
    }
  ],
}
~~~~
- This is the bare minimum configuration required for a storage account. You can use these [docs](https://docs.microsoft.com/en-au/azure/templates/microsoft.storage/2019-04-01/storageaccounts) to see what else is possible.
- Now that we have our template defined, and referenced in our **azure-pipelines.yml** we can commit and push to our repo to trigger the build and deployment.
- ***NOTE:*** The storage account must be unique, so make sure you name it accordingly or the pipeline will fail.
- Nagivate to the Azure Portal to see your deployments, or use this command to see all storage accounts:
~~~~
az storage account list
~~~~
- You should see your storage account listed under the resource group for that you created the service connection under.