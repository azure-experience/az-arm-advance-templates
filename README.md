# az-arm-advance-templates
A detailed run on few of the advance topics of ARM

## Index for different use-cases
### [1. Brief Introduction _(Before you start)_](#before-we-start)
### [2. Multi-resource creation](#multi-resource-creation)
### [3. Resource creation with parameters](#resource-creation-parameters)
### [4. Resource creation with functions](#resource-creation-functions)
### [5. Resource creation with variables](#resource-creation-variables)
### [6. How to get output for your ARM templates?](#arm-generate-output)
### [7. Usage of exported templates](#exported-templates)
### [8. Usage of quick start templates](#quick-start-templates)
### [9. Resource creation with tags](#resource-creation-tags)

--
--
--
--
--
--
--
### <a name="before-we-start"></a>1. Brief Introduction (_Before you start_)
To push these ARM templates to our Azure environment, we will rely on **PowerShell** & have the following **Azure Subscription details**, ready before we kick-off. 

:pushpin: **Pointer:**

I have shown the simulations for my subscription name. Feel free to use your subscription too :blush:

|Property|Definition|
|---|---|
|**Azure Subscription**|Pay-as-you-go|
|**Subscription Name**|_nags-azure-subscription_|
|**Resource Group Name**|_azure-lab-rg-01_|
|**Powershell Cmdlet**|_New-AzResourceGroupDeployment_|

Check your Powershell environment:

```
PS C:\WINDOWS\system32> (Get-AzContext).Subscription.Name
nags-azure-subscription

PS C:\WINDOWS\system32> (Get-AzContext).Subscription.id
XXXXXXXXX-YYYY-ABCD-EFGH-ABCDEFXYZ722

PS C:\WINDOWS\system32> (Get-AzContext).Subscription.State
Enabled
```

Before we create a new resource (within our resource group) using: _New-AzResourceGroupDeployment_, its advisable to familiarize with this cmdlet. You can choose its help function like given below.

```
PS C:\WINDOWS\system32> help New-AzResourceGroupDeployment -ShowWindow
```

### <a name="multi-resource-creation"></a>2. Multi-resource creation
|Property|Definition|
|---|---|
|Folder|[2-resource-creation](./2-resource-creation)|
|File|_azuredeploy.json_|

Run this command to create a **storageaccount** & **virtualnetwork** simultaneously in your resource group. Here you need to understand the usage of **positional elements within array**, and the mechanism to invoke it.

Ex: in the addressSpace parameter, there are 2 array fields: _addressPrefixes & subnets_ defined, where each of them have multiple elements (_firstvnetPrefix & secondvnetPrefix; firstSubnet & secondSubnet_). 

Observe how these elements are referred during the creation of Vnet resources (ex for address space parameter, to extract the addressprefix of _firstvnetPrefix_, the reference call is _"[parameters('allAddrSpaces').addressPrefixes[0].addressPrefix]"_.

**Extract from the json file:**
```
/* parameter definition */
...
"allAddrSpaces":{
      "type": "object",
      "defaultValue": {
          /* "name":"VNet1", (not mandatory to mention the profile name) */
            "addressPrefixes": [
                {
                    "name": "firstvnetPrefix",
                    "addressPrefix": "10.0.0.0/16"
                },
                {
                    "name": "secondvnetPrefix",
                    "addressPrefix": "10.0.1.0/16"
                }
            ],
            "subnets":[
                {
                    "name": "firstSubnet",
                    "addressPrefix": "10.0.0.0/24"
                },
                {
                    "name":"secondSubnet",
                    "addressPrefix":"10.0.1.0/24"
                }
            ]
        }
    },
...

/* resource definition */
...
{
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2019-11-01",
      "name": "[variables('vnetName')]",
      "tags": "[parameters('resourceTags')]",
      "location": "[parameters('location')]",
      "properties": {
          "addressSpace": {
            /* "addressPrefixes": "[parameters('vnetAddrSpace')]" */
            /* /* "type": "string", (not a string, but an object)*/
            "addressPrefixes": ["[parameters('allAddrSpaces').addressPrefixes[0].addressPrefix]", "[parameters('allAddrSpaces').addressPrefixes[1].addressPrefix]"]
            },
          "subnets": [
            {
              "name": "[parameters('allAddrSpaces').subnets[0].name]",
              "properties": {
                  "addressPrefix": "[parameters('allAddrSpaces').subnets[0].addressPrefix]",
                  "delegations": [],
                  "privateEndpointNetworkPolicies": "Enabled",
                  "privateLinkServiceNetworkPolicies": "Enabled"
                }
              },
...
```


**Command:**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\0-storage-vnet-creation> 
New-AzResourceGroupDeployment -Name "storagevnetcreate" -ResourceGroupName "azure-lab-rg-01" \
-TemplateFile .\azuredeploy.json -TemplateParameterFile .\azuredeploy.parameters.dev.json -Verbose
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 19:35:58 - Template is valid.
VERBOSE: 19:36:01 - Create template deployment 'storagevnetcreate'
VERBOSE: 19:36:06 - Resource Microsoft.Storage/storageAccounts 'dev12bxmueijtaz47c' provisioning status is running
VERBOSE: 19:36:06 - Resource Microsoft.Network/virtualNetworks 'dev12' provisioning status is running
VERBOSE: 19:36:22 - Resource Microsoft.Network/virtualNetworks 'dev12' provisioning status is succeeded
VERBOSE: 19:36:27 - Resource Microsoft.Storage/storageAccounts 'dev12bxmueijtaz47c' provisioning status is succeeded


DeploymentName          : storagevnetcreate
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 19-04-2020 14:06:28
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name               Type                       Value
                          =================  =========================  ==========
                          mystoragePrefix    String                     dev12
                          storageSKU         String                     Standard_LRS
                          location           String                     southindia
                          vnet_name          String                     dev12
                          allAddrSpaces      Object                     {
                            "addressPrefixes": [
                              {
                                "name": "firstPrefix",
                                "addressPrefix": "10.7.0.0/16"
                              },
                              {
                                "name": "secondPrefix",
                                "addressPrefix": "10.8.0.0/16"
                              }
                            ],
                            "subnets": [
                              {
                                "name": "subnet-dev-A",
                                "addressPrefix": "10.7.0.0/24"
                              },
                              {
                                "name": "subnet-dev-B",
                                "addressPrefix": "10.8.0.0/24"
                              }
                            ]
                          }
                          resourceTags       Object                     {
                            "environment": "dev",
                            "lab-simulation": "arm-template-creation-dev"
                          }

Outputs                 :
                          Name               Type                       Value
                          =================  =========================  ==========
                          storageEndpoint    Object                     {
                            "services": {
                              "file": {
                                "enabled": true,
                                "lastEnabledTime": "2020-04-19T14:06:06.1041607Z"
                              },
                              "blob": {
                                "enabled": true,
                                "lastEnabledTime": "2020-04-19T14:06:06.1041607Z"
                              }
                            },
                            "keySource": "Microsoft.Storage"
                          }
                          storageStatus      String                     available
                          creationTime       String                     19-04-2020 14:06:06
                          vnetAddrSpace      Object                     {
                            "addressPrefixes": [
                              "10.7.0.0/16",
                              "10.8.0.0/16"
                            ]
                          }
                          vnetSubnet         Array                      [
                            {
                              "name": "subnet-dev-A",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/dev12/subnets/subnet-dev-A",
                              "etag": "W/\"8b49ab4f-2ecf-4f9c-baf7-9813b12792d0\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.7.0.0/24",
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            },
                            {
                              "name": "subnet-dev-B",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/dev12/subnets/subnet-dev-B",
                              "etag": "W/\"8b49ab4f-2ecf-4f9c-baf7-9813b12792d0\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.8.0.0/24",
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            }
                          ]
                          vnetPeeringInfo    Array                      []

DeploymentDebugLogLevel :
```
