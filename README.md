# ARM templates

## Deployment modes
- ***Incremental update (DEFAULT):*** Azure Resource Manager only makes changes to existing resources if they are specified in the ARM template. Pre-existing resources that are not referenced in the ARM template are left alone.  

- ***Complete update:*** Azure Resource Manager will delete all resources within a resource group that are not referenced in the ARM template. Essentially, the ARM template will reflect the current state of all the resources within a resource group.