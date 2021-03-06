# az-arm-advance-templates
A detailed run on few of the **advance topics of ARM, used with Azure Resource Manager templates & PowerShell.** It includes

1. how to create a **complex az resource** which has multi-dependency with other az resources?

2. how to not build everything from scratch, and rely on **pre-existing templates**?

3. how to **secure your az resources** (right from its creation till the deployment)?

and **many more..!**

:mega: **Key Assumption(s) at this point:**

a. This course has been built, bearing in mind that **you are aware of the ARM terminologies** (ex: _parameter, resource, variable, functions, etc_). If you are not aware, then please feel free to indulge in the [basic refresher course](https://github.com/azure-experience/az-arm-basic-templates)

b. It also assumes that you have basic capability on **Windows PowerShell**, and esp on the **subscription/environ related az cmdlets** (ex: _get-azsubscription, get-azcontext, set-azcontext, etc_)

Hope this course gives you the much needed info you are looking for :smiley:

---

## Index for different use-cases
### [1. Brief Introduction _(Before you start)_](#before-we-start)
### [2. Multi-resource creation](#multi-resource-creation)
### [3. Deploy a pre-defined template](#pre-defined-template)
### [4. Concurrent resource deployment](#concurrent-resource-creation)
### [5. Resource inter-dependency](#resource-inter-dependency)
### [6. Usage of deployment scripts](#deployment-scripts)
### [7. Conditional-based deployment](#conditional-deployment)
### [8. Securitize credentials with Keyvault](#keyvault-for-credentials)
### [9. Usage of Linked templates](#linked-templates)
### [10. Usage of VM extensions](#vm-extensions)

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
|Folder|[0-storage-vnet-creation](./0-storage-vnet-creation)|
|File|_azuredeploy.json_|
|ParameterFile|_azuredeploy.parameters.dev.json_|

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

### <a name="pre-defined-template"></a>3. Deploy a pre-defined template
|Property|Definition|
|---|---|
|Folder|[1-quick-template-reference](./1-quick-template-reference)|
|File|_azuredeploy.json_|

We can always rely on the ["pre-defined" templates](https://github.com/Azure/azure-quickstart-templates) already provided for Azure resources (within github). It could save you a lot of configuration time and speed up new resource creation for any az service.

![](imgs/a-github-quick-start-az-templates.png)

**Command:**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\1-quick-template-reference> 
New-AzResourceGroupDeployment -Name "quicktemplateusage" -ResourceGroupName "azure-lab-rg-01" \
-TemplateFile .\azuredeploy.json -Verbose
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 19:40:46 - Template is valid.
VERBOSE: 19:40:49 - Create template deployment 'quicktemplateusage'
VERBOSE: 19:40:54 - Resource Microsoft.Storage/storageAccounts 'azstoragebxmueijtaz47c' provisioning status is running
VERBOSE: 19:41:18 - Resource Microsoft.Storage/storageAccounts 'azstoragebxmueijtaz47c' provisioning status is succeeded

DeploymentName          : quicktemplateusage
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 19-04-2020 14:11:18
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                    Type                       Value
                          ======================  =========================  ==========
                          storageReferenceName    String                     azstorage
                          storageAccountType      String                     Standard_LRS
                          location                String                     southindia

Outputs                 :
                          Name                       Type                       Value
                          =========================  =========================  ==========
                          stgaccounstgnetworkacls    Object                     {
                            "bypass": "AzureServices",
                            "virtualNetworkRules": [],
                            "ipRules": [],
                            "defaultAction": "Allow"
                          }
                          stgencryption              Object                     {
                            "services": {
                              "file": {
                                "enabled": true,
                                "lastEnabledTime": "2020-04-19T14:10:54.7900009Z"
                              },
                              "blob": {
                                "enabled": true,
                                "lastEnabledTime": "2020-04-19T14:10:54.7900009Z"
                              }
                            },
                            "keySource": "Microsoft.Storage"
                          }
                          stgccesstier               String                     Hot
                          stglocation                String                     southindia
                          stgendpoints               Object                     {
                            "dfs": "https://azstoragebxmueijtaz47c.dfs.core.windows.net/",
                            "web": "https://azstoragebxmueijtaz47c.z30.web.core.windows.net/",
                            "blob": "https://azstoragebxmueijtaz47c.blob.core.windows.net/",
                            "queue": "https://azstoragebxmueijtaz47c.queue.core.windows.net/",
                            "table": "https://azstoragebxmueijtaz47c.table.core.windows.net/",
                            "file": "https://azstoragebxmueijtaz47c.file.core.windows.net/"
                          }

DeploymentDebugLogLevel :
```

### <a name="concurrent-resource-creation"></a>4. Concurrent resource deployment
|Property|Definition|
|---|---|
|Folder|[2-multiple-concurrent-copies](./2-multiple-concurrent-copies)|
|File|_azuredeploy.json_|

Sometimes there are scenarios, where you might need to **deploy multiple, identical copy of az resources**, which can be accomplished by usage of _copyIndex_ function. Below we are trying to **create 3 storage accounts of identical configuration** in the same region. 

**Extract from the json file:**
```
/* resource definition */
...
"resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2019-04-01",
      // dont need to reference any variable as its already commented out
      // "name": "[variables('storageAccountName')]",
      "name": "[concat(copyIndex(),'azstorage', uniqueString(resourceGroup().id))]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[parameters('storageAccountType')]"
      },
      "kind": "StorageV2",
      "copy": {
          "name": "azstoragecopy",
          "count": 3
        },
...
```

:pushpin: **Key-pointer(s):**

When you are trying to create identical copy of the az resources, **please dont declare a parameter or variable explicitly**, as it works well if the resource name is directly coded during its creation.

**Command:**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\2-multiple-concurrent-copies> 
New-AzResourceGroupDeployment -Name "multipleresourcecreation" -ResourceGroupName "azure-lab-rg-01" \
-TemplateFile .\azuredeploy.json -Verbose
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 19:49:08 - Template is valid.
VERBOSE: 19:49:10 - Create template deployment 'multipleresourcecreation'
VERBOSE: 19:49:16 - Resource Microsoft.Storage/storageAccounts '0azstoragebxmueijtaz47c' provisioning status is running
VERBOSE: 19:49:35 - Resource Microsoft.Storage/storageAccounts '1azstoragebxmueijtaz47c' provisioning status is running
VERBOSE: 19:49:35 - Resource Microsoft.Storage/storageAccounts '2azstoragebxmueijtaz47c' provisioning status is running
VERBOSE: 19:49:40 - Resource Microsoft.Storage/storageAccounts '1azstoragebxmueijtaz47c' provisioning status is succeeded
VERBOSE: 19:49:40 - Resource Microsoft.Storage/storageAccounts '2azstoragebxmueijtaz47c' provisioning status is succeeded
VERBOSE: 19:49:40 - Resource Microsoft.Storage/storageAccounts '0azstoragebxmueijtaz47c' provisioning status is succeeded

DeploymentName          : multipleresourcecreation
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 19-04-2020 14:19:41
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                    Type                       Value
                          ======================  =========================  ==========
                          storageReferenceName    String                     azstorage
                          storageAccountType      String                     Standard_LRS
                          location                String                     southindia

Outputs                 :
DeploymentDebugLogLevel :
```

### <a name="resource-inter-dependency"></a>5. Resource Inter-dependency
|Property|Definition|
|---|---|
|Folder|[3-resource-inter-dependency](./3-resource-inter-dependency)|
|File|_azuredeploy.json_|
|ParameterFile|_azuredeploy.parameters.json_|

At times, there will be a need to create a complex azure resource which interfaces (either directly or indirectly) with other az resources. Its important to know the **"sequential-order-of-creation"** to help create first the independent resource & then the dependent resource for the right execution.

For ex: look at creation of a simple VM (see diagram below). 

a. **VM Creation** depends on 2 main components: _Storage Account_ & _Network-Interface-Card_

b. NIC itself depends on _Public IP Address & Virtual Network_ to be ready & online before it can be instantiated

c. And lastly VNet depends entirely on a uniform "incoming/outgoing traffic" ruleset, namely: _Network Security Groups"_ which can be safely attached to its Subnet during VM creation

![](imgs/b-vm-creation-workflow.png)

So its always advisable from the order of creation to

I. first **create independent components** (ex: _Storage account & Public IP Address_)

II. then **create "semi-independent" components**  with first layer of dependency (ex: _Vnet & NSG_)

III. and lastly, after all the other components (ex: _Storage account, Public IP Addr, Vnet, NSG_) are in place, you can safely create the NIC & then eventually the VM

:pushpin: **Key Pointer(s):**

1. Before proceeding to **create a multi-coupled resource**, please make **a simple flowchart of dependent & independent components**, as it helps in prioritization

2. Secondly, its recommended to attach NSG to subnet to homogenize the traffic security of VM's

3. And lastly, this VM creation process can be further extended to add the next set of components such as **PS Desired Confiuguration/Custom Script extension**

**Extract from the json file:**

Please see the resource definition for the **NSG** defined here [it allows incoming RDP connections to that VM (port _3389_)].
```
/* resource definition *?
...
{
    "type": "Microsoft.Network/networkSecurityGroups",
    "apiVersion": "2019-11-01",
    "name": "[variables('nsgName')]",
    "tags": "[parameters('resourceTags')]",
    "location": "[parameters('location')]",
    "properties": {
      "securityRules": [
        {
          "name": "inbound-rdp-rules",
          "properties": {
            "description": "default nsg template for nic",
            "protocol": "tcp",
            "access": "Allow",
            "priority": "100",
            "direction": "Inbound",
            "sourcePortRange": "*",
            "destinationPortRange": "3389",
            "sourceAddressPrefix": "*",
            "destinationAddressPrefix": "VirtualNetwork"
          }
        }
      ]
    }
  },
  ...
  ```
  
Now check the resource definition for **VNet**, which directly depends on the NSG to be pre-created & ready. Vnet resource references the **resourceId for the NSG** (which is available after its creation). It relies on the usage of **dependsOn** method within a resource definition.

```
/* parameter definition */
...
{
    "type": "Microsoft.Network/virtualNetworks",
    "apiVersion": "2019-11-01",
    "name": "[variables('vnetName')]",
    "tags": "[parameters('resourceTags')]",
    "location": "[parameters('location')]",
    "dependsOn": [
        "[resourceId('Microsoft.Network/networkSecurityGroups',variables('nsgName'))]"
    ],
...
```

Another example of resource dependency for VM, which relies on **Storage account & NIC** (via its **resourceId**, of course).

```
/* resource definition */
...
{
    "type": "Microsoft.Compute/virtualMachines",
    "apiVersion": "2019-07-01",
    "name": "[variables('vmName')]",
    "location": "[parameters('location')]",
    "tags": "[parameters('resourceTags')]",
    "dependsOn":[
      "[resourceId('Microsoft.Storage/storageAccounts',variables('storageAccountName'))]",
      "[resourceId('Microsoft.Network/networkInterfaces',variables('nicName'))]"
    ],
 ...
 ```
 
 **Command:**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\3-resource-inter-dependency> 
New-AzResourceGroupDeployment -Name "resourceinterdependency" -ResourceGroupName "azure-lab-rg-01" \
-TemplateFile .\azuredeploy.json -TemplateParameterFile .\azuredeploy.parameters.json -Verbose
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 20:22:10 - Template is valid.
VERBOSE: 20:22:12 - Create template deployment 'resourceinterdependency'
VERBOSE: 20:22:18 - Resource Microsoft.Network/publicIPAddresses 'aznewipbxmueijtaz47c' provisioning status is running
VERBOSE: 20:22:18 - Resource Microsoft.Network/networkSecurityGroups 'aznewnsgbxmueijtaz47c' provisioning status is running
VERBOSE: 20:22:18 - Resource Microsoft.Storage/storageAccounts 'aznewstgbxmueijtaz47c' provisioning status is running
VERBOSE: 20:22:35 - Resource Microsoft.Network/networkInterfaces 'aznewnicbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 20:22:35 - Resource Microsoft.Network/virtualNetworks 'aznewvnetbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 20:22:35 - Resource Microsoft.Network/publicIPAddresses 'aznewipbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 20:22:35 - Resource Microsoft.Network/networkSecurityGroups 'aznewnsgbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 20:22:41 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is running
VERBOSE: 20:22:41 - Resource Microsoft.Storage/storageAccounts 'aznewstgbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 20:23:13 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is succeeded

DeploymentName          : resourceinterdependency
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 19-04-2020 14:53:12
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                          Type                       Value
                          ============================  =========================  ==========
                          uniqueName                    String                     aznew
                          storagereplicationstrategy    String                     Standard_LRS
                          location                      String                     southindia
                          dnslabelprefix                String                     mynewapp
                          allAddrSpaces                 Object                     {
                            "addressPrefixes": [
                              {
                                "name": "firstvnetPrefix",
                                "addressPrefix": "10.0.0.0/16"
                              },
                              {
                                "name": "secondvnetPrefix",
                                "addressPrefix": "10.1.0.0/16"
                              }
                            ],
                            "subnets": [
                              {
                                "name": "firstSubnet",
                                "addressPrefix": "10.0.0.0/24"
                              },
                              {
                                "name": "secondSubnet",
                                "addressPrefix": "10.1.0.0/24"
                              }
                            ]
                          }
                          sizeofVM                      String                     Standard_DS1_v2
                          windowsImageSKU               String                     2019-Datacenter
                          computerName                  String                     windowsvm
                          adminAccount                  String                     azureadmin
                          adminPassword                 String                     Azurevm@4590
                          resourceTags                  Object                     {
                            "environment": "simulation",
                            "lab-simulation": "arm-template-inter-dependency-exercise"
                          }

Outputs                 :
                          Name                 Type                       Value
                          ===================  =========================  ==========
                          storageEndpoint      Object                     {
                            "dfs": "https://aznewstgbxmueijtaz47c.dfs.core.windows.net/",
                            "web": "https://aznewstgbxmueijtaz47c.z30.web.core.windows.net/",
                            "blob": "https://aznewstgbxmueijtaz47c.blob.core.windows.net/",
                            "queue": "https://aznewstgbxmueijtaz47c.queue.core.windows.net/",
                            "table": "https://aznewstgbxmueijtaz47c.table.core.windows.net/",
                            "file": "https://aznewstgbxmueijtaz47c.file.core.windows.net/"
                          }
                          storageencryption    Object                     {
                            "services": {
                              "file": {
                                "enabled": true,
                                "lastEnabledTime": "2020-04-19T14:52:17.3247491Z"
                              },
                              "blob": {
                                "enabled": true,
                                "lastEnabledTime": "2020-04-19T14:52:17.3247491Z"
                              }
                            },
                            "keySource": "Microsoft.Storage"
                          }
                          publicIPAddress      String                     104.211.211.58
                          publicdnsSettings    Object                     {
                            "domainNameLabel": "mynewapp",
                            "fqdn": "mynewapp.southindia.cloudapp.azure.com"
                          }
                          vnetAddrSpace        Object                     {
                            "addressPrefixes": [
                              "10.0.0.0/16",
                              "10.1.0.0/16"
                            ]
                          }
                          vnetSubnet           Array                      [
                            {
                              "name": "firstSubnet",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/aznewvnetbxmueijtaz47c/subnets/firstSubnet",
                              "etag": "W/\"6de58d6c-eaa3-4357-ab7b-acbf7463044a\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.0.0.0/24",
                                "networkSecurityGroup": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkSecurityGroups/aznewnsgbxmueijtaz47c"
                                },
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            },
                            {
                              "name": "secondSubnet",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/aznewvnetbxmueijtaz47c/subnets/secondSubnet",
                              "etag": "W/\"6de58d6c-eaa3-4357-ab7b-acbf7463044a\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.1.0.0/24",
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            }
                          ]
                          nsgInfo              Array                      [
                            {
                              "name": "inbound-rdp-rules",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkSecurityGroups/aznewnsgbxmueijtaz47c/securityRules/inbound-rdp-rules",
                              "etag": "W/\"cf0f8639-26cd-4844-9bf3-69095bb914da\"",
                              "type": "Microsoft.Network/networkSecurityGroups/securityRules",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "description": "default nsg template for nic",
                                "protocol": "tcp",
                                "sourcePortRange": "*",
                                "destinationPortRange": "3389",
                                "sourceAddressPrefix": "*",
                                "destinationAddressPrefix": "VirtualNetwork",
                                "access": "Allow",
                                "priority": 100,
                                "direction": "Inbound",
                                "sourcePortRanges": [],
                                "destinationPortRanges": [],
                                "sourceAddressPrefixes": [],
                                "destinationAddressPrefixes": []
                              }
                            }
                          ]
                          nicInfo              Array                      [
                            {
                              "name": "ipconfig1",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkInterfaces/aznewnicbxmueijtaz47c/ipConfigurations/ipconfig1",
                              "etag": "W/\"bd06f0e2-ccd0-48ad-88d5-5720b30ff5c6\"",
                              "type": "Microsoft.Network/networkInterfaces/ipConfigurations",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "privateIPAddress": "10.0.0.4",
                                "privateIPAllocationMethod": "Dynamic",
                                "publicIPAddress": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/publicIPAddresses/aznewipbxmueijtaz47c"
                                },
                                "subnet": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/aznewvnetbxmueijtaz47c/subnets/firstSubnet"
                                },
                                "primary": true,
                                "privateIPAddressVersion": "IPv4"
                              }
                            }
                          ]
                          vmInfo               Object                     {
                            "vmId": "4325d8ee-786f-4f20-a89f-87cc2bfda7ed",
                            "hardwareProfile": {
                              "vmSize": "Standard_DS1_v2"
                            },
                            "storageProfile": {
                              "imageReference": {
                                "publisher": "MicrosoftWindowsServer",
                                "offer": "WindowsServer",
                                "sku": "2019-Datacenter",
                                "version": "latest",
                                "exactVersion": "17763.1158.2004131759"
                              },
                              "osDisk": {
                                "osType": "Windows",
                                "name": "aznewvmbxmueijtaz47c_disk1_e8eb3f1a82004f1392b97436448c39c4",
                                "createOption": "FromImage",
                                "caching": "ReadWrite",
                                "managedDisk": {
                                  "storageAccountType": "Premium_LRS",
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/AZURE-LAB-RG-01/providers/Microsoft.Compute/disks/aznewvmbxmueijtaz47c_disk1_e8eb3f1a82004f1392b97436448c39c4"
                                },
                                "diskSizeGB": 127
                              },
                              "dataDisks": []
                            },
                            "osProfile": {
                              "computerName": "windowsvm",
                              "adminUsername": "azureadmin",
                              "windowsConfiguration": {
                                "provisionVMAgent": true,
                                "enableAutomaticUpdates": true
                              },
                              "secrets": [],
                              "allowExtensionOperations": true,
                              "requireGuestProvisionSignal": true
                            },
                            "networkProfile": {
                              "networkInterfaces": [
                                {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkInterfaces/aznewnicbxmueijtaz47c"
                                }
                              ]
                            },
                            "diagnosticsProfile": {
                              "bootDiagnostics": {
                                "enabled": true,
                                "storageUri": "https://aznewstgbxmueijtaz47c.blob.core.windows.net/"
                              }
                            },
                            "provisioningState": "Succeeded"
                          }

DeploymentDebugLogLevel :
```

### <a name="deployment-scripts"></a>6. Usage of deployment scripts

Deployment Scripts can be used in scenarios, where the custom steps are not possible due to limitations with ARM templates. As this feature is in preview mode (as on _Apr 24, 2020_), i havent dwelled in-depth on this topic.

More notes on this topic will be performed and documented soon!

### <a name="conditional-deployment"></a>7. Conditional-based deployment
|Property|Definition|
|---|---|
|Folder|[5-use-conditions](./5-use-conditions)|
|File|_azuredeploy.json_|
|ParameterFile|_azuredeploy.parameters.json_|

Sometimes, while creating multi-coupled resources, there comes a time when we can choose to create a resource **based on a certain criteria.** Criteria could range from

a. create a resource when a certain "flag/signal" is invoked **(or)**

b. create a resource subject to its pre-availability

In such scenarios, it becomes quite handy to rely on **conditional resource deployment**, where you specify a condition under which a resource deployment call is triggered. Its highly advisable to use this option to **spin up only independent az resources within that multi-coupled resource deployment**, as the behavior of your deployment will be much more predictable. (i.e. no sporadic dependency failures).

There are **2 steps** involved in using condition-based deployment

1. define **a parameter** to capture the flags pertaining to usage of a condition (ex: parameter could be _"creationCondition"_, and flags could be _"yes"_ or _"no"_)

2. consume this condition during the **resource definition** within the ARM template

**Extract from the json file:**
```
/* parameter definition *?
...
"conditionParamsforStorage": {
      "type": "string",
      "allowedValues": [
          "new",
          "existing"
      ]
    },
...

/* resource definition *?
...
{
    "condition": "[equals(parameters('conditionParamsforStorage'),'new')]",
    "type": "Microsoft.Storage/storageAccounts",
    "apiVersion": "2019-04-01",
    "name": "[parameters('storageAccountName')]",
    "tags": "[parameters('resourceTags')]",
    "location": "[parameters('location')]",
    "sku": {
      "name": "[parameters('storagereplicationstrategy')]"
    },
...
```

**Command:**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\5-use-conditions> 
New-AzResourceGroupDeployment -Name "conditionaldeployment" -ResourceGroupName "azure-lab-rg-01" \
-TemplateFile .\azuredeploy.json -TemplateParameterFile .\azuredeploy.parameters.json \
-storageAccountName "storage09az" -conditionParamsforStorage "new" -Verbose
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 20:46:40 - Template is valid.
VERBOSE: 20:46:42 - Create template deployment 'conditionaldeployment'
VERBOSE: 20:46:47 - Resource Microsoft.Network/publicIPAddresses 'aznewipbxmueijtaz47c' provisioning status is running
VERBOSE: 20:46:47 - Resource Microsoft.Storage/storageAccounts 'storage09az' provisioning status is running
VERBOSE: 20:46:47 - Resource Microsoft.Network/networkSecurityGroups 'aznewnsgbxmueijtaz47c' provisioning status is running
VERBOSE: 20:47:04 - Resource Microsoft.Network/networkInterfaces 'aznewnicbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 20:47:04 - Resource Microsoft.Network/virtualNetworks 'aznewvnetbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 20:47:04 - Resource Microsoft.Network/publicIPAddresses 'aznewipbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 20:47:04 - Resource Microsoft.Network/networkSecurityGroups 'aznewnsgbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 20:47:10 - Resource Microsoft.Storage/storageAccounts 'storage09az' provisioning status is succeeded
VERBOSE: 20:47:15 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is running
VERBOSE: 20:47:46 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is succeeded

DeploymentName          : conditionaldeployment
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 19-04-2020 15:17:48
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                          Type                       Value
                          ============================  =========================  ==========
                          uniqueName                    String                     aznew
                          storageAccountName            String                     storage09az
                          conditionParamsforStorage     String                     new
                          storagereplicationstrategy    String                     Standard_LRS
                          location                      String                     southindia
                          dnslabelprefix                String                     mynewapp
                          allAddrSpaces                 Object                     {
                            "addressPrefixes": [
                              {
                                "name": "firstvnetPrefix",
                                "addressPrefix": "10.0.0.0/16"
                              },
                              {
                                "name": "secondvnetPrefix",
                                "addressPrefix": "10.1.0.0/16"
                              }
                            ],
                            "subnets": [
                              {
                                "name": "firstSubnet",
                                "addressPrefix": "10.0.0.0/24"
                              },
                              {
                                "name": "secondSubnet",
                                "addressPrefix": "10.1.0.0/24"
                              }
                            ]
                          }
                          sizeofVM                      String                     Standard_DS1_v2
                          windowsImageSKU               String                     2019-Datacenter
                          computerName                  String                     windowsvm
                          adminAccount                  String                     azureadmin
                          adminPassword                 String                     Azurevm@4590
                          resourceTags                  Object                     {
                            "environment": "simulation",
                            "lab-simulation": "arm-template-condition-based-resource-creation"
                          }

Outputs                 :
                          Name                 Type                       Value
                          ===================  =========================  ==========
                          publicIPAddress      String                     104.211.209.218
                          publicdnsSettings    Object                     {
                            "domainNameLabel": "mynewapp",
                            "fqdn": "mynewapp.southindia.cloudapp.azure.com"
                          }
                          vnetAddrSpace        Object                     {
                            "addressPrefixes": [
                              "10.0.0.0/16",
                              "10.1.0.0/16"
                            ]
                          }
...
/* output truncated as its similar to the previous command run */
```

### <a name="keyvault-for-credentials"></a>8. Securitize credentials with Keyvault
|Property|Definition|
|---|---|
|Folder|[6-keyvault-for-secret-parameters](./6-keyvault-for-secret-parameters)|
|KeyVauult File|_azuredeploy-keyvault.json_|
|File|_azuredeploy.json_|
|ParameterFile|_azuredeploy.parameters.json_|

While creating VM' or any az resource, there is always a need to **secure your credentials**, such that they are not exposed either during the _creation, transit or deployed_ stage. Relying on **Azure Keyvault** becomes the absolute key.

In this example (for VM creation), we try to 

a. first create a **az keyvault**, and store the **VM's credentials** safely inside the keyvault

b. and lastly, **inject the secure password** (from _az keyvault_) directly **as a parameter** into the VM creation stage

So lets begin first with **creation of a keyvault**, and **build the secret credentials**

**Extract from the json file:**
```
/* parameter definition */
...
"secretName": {
      "type": "string",
      "defaultValue": "vmAdminPassword"
},
"secretValue": {
      "type": "securestring"
},
...

/* resource definition */
...
 {
    "type": "Microsoft.KeyVault/vaults/secrets",
    "apiVersion": "2018-02-14",
    // as the child: secret is outside of parent (kv) resource scope, it needs to be of format: parent/child
    "name": "[concat(variables('keyVaultName'), '/', parameters('secretName'))]",
    "tags": "[parameters('resourceTags')]",
    "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', variables('keyVaultName'))]"
    ],
    "properties": {
      "value": "[parameters('secretValue')]",
      "contentType": "string"
    }
  }
],
...
```

:pushpin: **Key Pointer(s):**

1. Here in the parameter definition, as we have declared `secretValue` of type: `"securestring"`, which gives us a chance from the console, **to inject a password, which will not be visible at the creation time**

2. Secondly, from the above parameter definition, you will notice that the _resource: */vaults/secrets_ creation is **actually a "child"** of the parent _resource: */vaults_, which is the reason for the direct-dependency to be there

**Command (for KeyVault creation):**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\6-keyvault-for-secret-parameters> 
New-AzResourceGroupDeployment -Name "keyvaultcreation" -ResourceGroupName "azure-lab-rg-01" \
-TemplateFile .\azuredeploy-keyvault.json -Verbose

cmdlet New-AzResourceGroupDeployment at command pipeline position 1
Supply values for the following parameters:
(Type !? for Help.)
secretValue: ************
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 20:53:08 - Template is valid.
VERBOSE: 20:53:10 - Create template deployment 'keyvaultcreation'
VERBOSE: 20:53:21 - Resource Microsoft.KeyVault/vaults 'azkvbxmueijtaz47c' provisioning status is running
VERBOSE: 20:53:38 - Resource Microsoft.KeyVault/vaults/secrets 'azkvbxmueijtaz47c/vmAdminPassword' provisioning status is succeeded
VERBOSE: 20:53:38 - Resource Microsoft.KeyVault/vaults 'azkvbxmueijtaz47c' provisioning status is succeeded

DeploymentName          : keyvaultcreation
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 19-04-2020 15:23:36
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name             Type                       Value
                          ===============  =========================  ==========
                          uniqueName       String                     az
                          location         String                     southindia
                          adUserId         String                     0164c813-8650-4e2f-b4bb-3d2d6de7bf2e
                          secretName       String                     vmAdminPassword
                          secretValue      SecureString
                          resourceTags     Object                     {
                            "environment": "simulation",
                            "lab-simulation": "arm-template-keyvault-exercise"
                          }

Outputs                 :
                          Name             Type                       Value
                          ===============  =========================  ==========
                          vaultUri         String                     https://azkvbxmueijtaz47c.vault.azure.net/

DeploymentDebugLogLevel :
```
--

Now after the **az keyvault** is created with **stored secret credentials**, its now time to build the parameter file to help **inject this very credentials** during the VM creation stage.

Lets navigate to the parameter file: _azuredeploy.parameters.json_, and observe this parameter section, where the **keyvault reference** is completed.

```
/* parameter definition */
...
"adminAccount": {
        "type": "string",
        "value": "azureadmin"
    },
"adminPassword": {
        "type": "securestring",
        "reference": {
        "keyVault": {
            "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg- \
                  01/providers/Microsoft.KeyVault/vaults/azkvbxmueijtaz47c"
        },
        "secretName": "vmAdminPassword"
      }
    }
...
```

In this file, we explicitly **declare the resourceId** of the Keyvault resource, and point directly to the `secretName` (i.e. _vmAdminPassword_) as created in the prior step.

Also you will notice that the **default login to this VM**: _azureadmin_.

Now, lets run the command to **create the VM, in tact with the parameter files**.

**Command (for VM creation):**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\6-keyvault-for-secret-parameters> 
New-AzResourceGroupDeployment -Name "keyvaultintegration" -ResourceGroupName "azure-lab-rg-01" \
-TemplateFile .\azuredeploy.json -TemplateParameterFile .\azuredeploy.parameters.json \
-storageAccountName "azold" -conditionParamsforStorage "new" -Verbose
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 21:02:42 - Template is valid.
VERBOSE: 21:02:45 - Create template deployment 'keyvaultintegration'
VERBOSE: 21:02:51 - Resource Microsoft.Network/publicIPAddresses 'aznewipbxmueijtaz47c' provisioning status is running
VERBOSE: 21:02:51 - Resource Microsoft.Storage/storageAccounts 'azold' provisioning status is running
VERBOSE: 21:02:51 - Resource Microsoft.Network/networkSecurityGroups 'aznewnsgbxmueijtaz47c' provisioning status is running
VERBOSE: 21:03:07 - Resource Microsoft.Network/networkInterfaces 'aznewnicbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 21:03:07 - Resource Microsoft.Network/virtualNetworks 'aznewvnetbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 21:03:07 - Resource Microsoft.Network/publicIPAddresses 'aznewipbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 21:03:07 - Resource Microsoft.Network/networkSecurityGroups 'aznewnsgbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 21:03:13 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is running
VERBOSE: 21:03:14 - Resource Microsoft.Storage/storageAccounts 'azold' provisioning status is succeeded
VERBOSE: 21:03:51 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is succeeded

DeploymentName          : keyvaultintegration
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 19-04-2020 15:33:51
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                          Type                       Value
                          ============================  =========================  ==========
                          uniqueName                    String                     aznew
                          storageAccountName            String                     azold
                          conditionParamsforStorage     String                     new
                          storagereplicationstrategy    String                     Standard_LRS
                          location                      String                     southindia
                          dnslabelprefix                String                     mynewapp
                          allAddrSpaces                 Object                     {
                            "addressPrefixes": [
                              {
                                "name": "firstvnetPrefix",
                                "addressPrefix": "10.0.0.0/16"
                              },
                              {
                                "name": "secondvnetPrefix",
                                "addressPrefix": "10.1.0.0/16"
                              }
                            ],
                            "subnets": [
                              {
                                "name": "firstSubnet",
                                "addressPrefix": "10.0.0.0/24"
                              },
                              {
                                "name": "secondSubnet",
                                "addressPrefix": "10.1.0.0/24"
                              }
                            ]
                          }
                          sizeofVM                      String                     Standard_DS1_v2
                          windowsImageSKU               String                     2019-Datacenter
                          computerName                  String                     mywinvm
                          adminAccount                  String                     azureadmin
                          adminPassword                 String
                          resourceTags                  Object                     {
                            "environment": "simulation",
                            "lab-simulation": "arm-template-keyvault-secret-integration"
                          }

Outputs                 :
                          Name                 Type                       Value
                          ===================  =========================  ==========
                          publicIPAddress      String                     104.211.231.138
                          publicdnsSettings    Object                     {
                            "domainNameLabel": "mynewapp",
                            "fqdn": "mynewapp.southindia.cloudapp.azure.com"
                          }
                          vnetAddrSpace        Object                     {
                            "addressPrefixes": [
                              "10.0.0.0/16",
                              "10.1.0.0/16"
                            ]
                          }
                          vnetSubnet           Array                      [
                            {
                              "name": "firstSubnet",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/aznewvnetbxmueijtaz47c/subnets/firstSubnet",
                              "etag": "W/\"75e0fab1-ea3a-4b0f-8ede-0b2584ce7de6\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.0.0.0/24",
                                "networkSecurityGroup": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkSecurityGroups/aznewnsgbxmueijtaz47c"
                                },
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            },
                            {
                              "name": "secondSubnet",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/aznewvnetbxmueijtaz47c/subnets/secondSubnet",
                              "etag": "W/\"75e0fab1-ea3a-4b0f-8ede-0b2584ce7de6\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.1.0.0/24",
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            }
                          ]
                          nsgInfo              Array                      [
                            {
                              "name": "inbound-rdp-rules",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkSecurityGroups/aznewnsgbxmueijtaz47c/securityRules/inbound-rdp-rules",
                              "etag": "W/\"ea92794e-c08a-4b0e-81e0-7092c81c0e45\"",
                              "type": "Microsoft.Network/networkSecurityGroups/securityRules",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "description": "default nsg template for nic",
                                "protocol": "tcp",
                                "sourcePortRange": "*",
                                "destinationPortRange": "3389",
                                "sourceAddressPrefix": "*",
                                "destinationAddressPrefix": "VirtualNetwork",
                                "access": "Allow",
                                "priority": 100,
                                "direction": "Inbound",
                                "sourcePortRanges": [],
                                "destinationPortRanges": [],
                                "sourceAddressPrefixes": [],
                                "destinationAddressPrefixes": []
                              }
                            }
                          ]
                          nicInfo              Array                      [
                            {
                              "name": "ipconfig1",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkInterfaces/aznewnicbxmueijtaz47c/ipConfigurations/ipconfig1",
                              "etag": "W/\"895b64c5-3511-426c-b137-b9ae4f14223d\"",
                              "type": "Microsoft.Network/networkInterfaces/ipConfigurations",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "privateIPAddress": "10.0.0.4",
                                "privateIPAllocationMethod": "Dynamic",
                                "publicIPAddress": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/publicIPAddresses/aznewipbxmueijtaz47c"
                                },
                                "subnet": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/aznewvnetbxmueijtaz47c/subnets/firstSubnet"
                                },
                                "primary": true,
                                "privateIPAddressVersion": "IPv4"
                              }
                            }
                          ]
                          vmInfo               Object                     {
                            "vmId": "4368d2d9-aa72-45d7-a71a-f9a1e445f47d",
                            "hardwareProfile": {
                              "vmSize": "Standard_DS1_v2"
                            },
                            "storageProfile": {
                              "imageReference": {
                                "publisher": "MicrosoftWindowsServer",
                                "offer": "WindowsServer",
                                "sku": "2019-Datacenter",
                                "version": "latest",
                                "exactVersion": "17763.1158.2004131759"
                              },
                              "osDisk": {
                                "osType": "Windows",
                                "name": "aznewvmbxmueijtaz47c_disk1_89e7678964584a9b9a0474b910eaedc3",
                                "createOption": "FromImage",
                                "caching": "ReadWrite",
                                "managedDisk": {
                                  "storageAccountType": "Premium_LRS",
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/AZURE-LAB-RG-01/providers/Microsoft.Compute/disks/aznewvmbxmueijtaz47c_disk1_89e7678964584a9b9a0474b910eaedc3"
                                },
                                "diskSizeGB": 127
                              },
                              "dataDisks": []
                            },
                            "osProfile": {
                              "computerName": "mywinvm",
                              "adminUsername": "azureadmin",
                              "windowsConfiguration": {
                                "provisionVMAgent": true,
                                "enableAutomaticUpdates": true
                              },
                              "secrets": [],
                              "allowExtensionOperations": true,
                              "requireGuestProvisionSignal": true
                            },
                            "networkProfile": {
                              "networkInterfaces": [
                                {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkInterfaces/aznewnicbxmueijtaz47c"
                                }
                              ]
                            },
                            "diagnosticsProfile": {
                              "bootDiagnostics": {
                                "enabled": true,
                                "storageUri": "https://azold.blob.core.windows.net/"
                              }
                            },
                            "provisioningState": "Succeeded"
                          }

DeploymentDebugLogLevel :
```

After this VM creation is complete, you can **directly try to connect to this VM** (via RDP of course!), and then login to the VM with the username: _azureadmin_ & then enter the **same password** which you had typed for the `secretValue` during the keyvault creation. 

**You would be able to login without any hitch, (in a very secure mode of course!)**

### <a name="linked-templates"></a>9. Usage of Linked templates
|Property|Definition|
|---|---|
|Folder|[7-linked-templates](./7-linked-templates)|
|StorageAccount File|_azuredeploy.storageaccount.json_|
|File|_azuredeploy.json_|

This concept of using **linked templates**, is especially helpful, when you **have templatized a az resource, and have uploaded its template in an external site** (ex: could be in a **ftp server** or within a **blob (container)** in Azure), and its right then, you **wish to deploy a new resource, by referencing the same template.**

In this simulation, we have 

a. built a **template for storage account**: _azuredeploy.storageaccount.json_

b. created a storage account by name: _storagetemplatefolder_, based on that template

c. then proceed to **create a sample blob container**: _az-storage-template-folder_, and have uploaded this very template within this container. This will ensure that **for linked-based templates, the template can be accessed from this container**

b. and lastly, have proceeded to **create a complete VM** (including the **storage account from this template URL of the blob container**)

So lets begin first with **creation of a storage account & the subsequent blob (container)**

**Extract from the json file:**
```
/* parameter definition */
...
"parameters": {
    "storageAccountName": {
      "type": "string",
      "defaultValue": "storagetemplatefolder",
      "minLength": 3,
      "maxLength": 24
    },
...
```
**Command (for Storage Account creation):**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\7-linked-templates> 
New-AzResourceGroupDeployment -Name "storageaccountlink" \
-ResourceGroupName "azure-lab-rg-01" -TemplateFile .\azuredeploy.storageaccount.json \
-Verbose
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 19:42:45 - Template is valid.
VERBOSE: 19:42:47 - Create template deployment 'storageaccountlink'
VERBOSE: 19:42:58 - Resource Microsoft.Storage/storageAccounts 'storagetemplatefolder' provisioning status is running
VERBOSE: 19:43:17 - Resource Microsoft.Storage/storageAccounts 'storagetemplatefolder' provisioning status is succeeded

DeploymentName          : storageaccountlink
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 26-04-2020 14:13:13
Mode                    : Incremental
TemplateLink            : 
Parameters              : 
                          Name                          Type                       Value     
                          ============================  =========================  ==========
                          storageAccountName            String                     storagetemplatefolder
                          storagereplicationstrategy    String                     Standard_LRS
                          location                      String                     southindia
                          resourceTags                  Object                     {
                            "environment": "simulation",
                            "lab-simulation": "arm-template-linked-template"
                          }
                          
Outputs                 : 
                          Name                     Type                       Value     
                          =======================  =========================  ==========
                          storageEndpointBlob      String                     https://storagetemplatefolder.blob.core.windows.net/
                          storageencryptionBlob    Object                     {
                            "enabled": true,
                            "lastEnabledTime": "2020-04-26T14:12:51.9166378Z"
                          }
                          
DeploymentDebugLogLevel : 
```

After the storageAccount: **storagetemplatefolder** is created, proceed to create a blob (container) by name: _"az-storage-template-folder"_, and upload this template file: _azuredeploy.storageaccount.json_ within that blob.

Now at the last step, lets try to deploy the main json file: _azuredeploy.json_ which **encompasses the complete URL to the storage account template**.

**Extract from the json file:**
```
/* parameter definition */
...
"storageAccountLinkedTemplateURI": {
        "type": "string",
		/* preferred approach would be to use SAS token, instead of direct URL */
        "defaultValue": "https://storagetemplatefolder.blob.core.windows.net/az-storage-template- \
                        folder/azuredeploy.storageaccount.json"
},
...

/* variable definition */
...
"variables": {
    // drop the storage account name variable, as we rely on condition-based resource creation
    "storageAccountName": "[concat(parameters('uniqueName'), 'stg', uniquestring(resourceGroup().id))]",
...  

/* resource definition */
...
{
    "type": "Microsoft.Resources/deployments",
    "apiVersion": "2019-08-01",
    "name": "storageAccountLinkedTemplate",
    "tags": "[parameters('resourceTags')]",
    "properties": {
        "mode": "Incremental",
        "templateLink": {
            "uri": "[parameters('storageAccountLinkedTemplateURI')]"
        },
        "parameters": {
            "storageAccountName": {"value": "[variables('storageAccountName')]"},
            "location": {"value": "[parameters('location')]"},
            "storagereplicationstrategy": {"value": "[parameters('storagereplicationstrategy')]"},
            "resourceTags": {"value": "[parameters('resourceTags')]"}
        }
      }
    },
...
```

:pushpin: **Key Pointer(s):**

1. In the parameter definition, we have given the **absolute path to the template file**, and the permissions given on this container is of **blob** privilege. Instead of giving absolute path, for **enhanced security**, you can **generate SAS token for the template file**, and append the same in the URL section

2. A variable name: `storageAccountName` has also been defined to inject them during resource creation

3. and lastly, a resource of type: `"Microsoft.Resources/deployments"` is used to satisfy the **use-case of deploying a az resource based on a pre-available template**. 

4. **Its highly recommended to understand the parameters (for your az resource) that your template needs before calling this resource type**

Now, lets run the command to **create the VM, in tact with the parameter files.**

**Command (for VM creation):**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\7-linked-templates> 
New-AzResourceGroupDeployment -Name "linkedtemplatecreation" -ResourceGroupName "azure-lab-rg-01" \
-TemplateFile .\azuredeploy.json -Verbose
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 19:49:41 - Template is valid.
VERBOSE: 19:49:43 - Create template deployment 'linkedtemplatecreation'
VERBOSE: 19:49:49 - Resource Microsoft.Resources/deployments 'storageAccountLinkedTemplate' provisioning status is running
VERBOSE: 19:49:55 - Resource Microsoft.Network/virtualNetworks 'aznewvnetbxmueijtaz47c' provisioning status is running
VERBOSE: 19:49:55 - Resource Microsoft.Network/publicIPAddresses 'aznewipbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 19:49:55 - Resource Microsoft.Network/networkSecurityGroups 'aznewnsgbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 19:49:55 - Resource Microsoft.Storage/storageAccounts 'aznewstgbxmueijtaz47c' provisioning status is running
VERBOSE: 19:50:10 - Resource Microsoft.Network/networkInterfaces 'aznewnicbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 19:50:10 - Resource Microsoft.Network/virtualNetworks 'aznewvnetbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 19:50:22 - Resource Microsoft.Storage/storageAccounts 'aznewstgbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 19:50:27 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is running
VERBOSE: 19:50:27 - Resource Microsoft.Resources/deployments 'storageAccountLinkedTemplate' provisioning status is succeeded
VERBOSE: 19:52:43 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is succeeded

DeploymentName          : linkedtemplatecreation
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 26-04-2020 14:22:41
Mode                    : Incremental
TemplateLink            : 
Parameters              : 
                          Name                               Type                       Value     
                          =================================  =========================  ==========
                          uniqueName                         String                     aznew     
                          storageAccountLinkedTemplateURI    String                     
                          https://storagetemplatefolder.blob.core.windows.net/az-storage-template-folder/azuredeploy.storageaccount.json
                          storagereplicationstrategy         String                     Standard_LRS
                          location                           String                     southindia
                          dnslabelprefix                     String                     mynewapp  
                          allAddrSpaces                      Object                     {
                            "addressPrefixes": [
                              {
                                "name": "firstvnetPrefix",
                                "addressPrefix": "10.0.0.0/16"
                              },
                              {
                                "name": "secondvnetPrefix",
                                "addressPrefix": "10.1.0.0/16"
                              }
                            ],
                            "subnets": [
                              {
                                "name": "firstSubnet",
                                "addressPrefix": "10.0.0.0/24"
                              },
                              {
                                "name": "secondSubnet",
                                "addressPrefix": "10.1.0.0/24"
                              }
                            ]
                          }
                          sizeofVM                           String                     Standard_DS1_v2
                          windowsImageSKU                    String                     2019-Datacenter
                          computerName                       String                     mywinvm   
                          adminAccount                       String                     azureadmin
                          adminPassword                      String                     Azurevm@123
                          resourceTags                       Object                     {
                            "environment": "simulation",
                            "lab-simulation": "arm-template-linked-template"
                          }
                          
Outputs                 : 
                          Name                 Type                       Value     
                          ===================  =========================  ==========
                          publicIPAddress      String                     104.211.213.197
                          publicdnsSettings    Object                     {
                            "domainNameLabel": "mynewapp",
                            "fqdn": "mynewapp.southindia.cloudapp.azure.com"
                          }
                          vnetAddrSpace        Object                     {
                            "addressPrefixes": [
                              "10.0.0.0/16",
                              "10.1.0.0/16"
                            ]
                          }
                          vnetSubnet           Array                      [
                            {
                              "name": "firstSubnet",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks
                          /aznewvnetbxmueijtaz47c/subnets/firstSubnet",
                              "etag": "W/\"64425f79-c9a8-4c17-90ce-cff0a3867d78\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.0.0.0/24",
                                "networkSecurityGroup": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkSecu
                          rityGroups/aznewnsgbxmueijtaz47c"
                                },
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            },
                            {
                              "name": "secondSubnet",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks
                          /aznewvnetbxmueijtaz47c/subnets/secondSubnet",
                              "etag": "W/\"64425f79-c9a8-4c17-90ce-cff0a3867d78\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.1.0.0/24",
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            }
                          ]
                          nsgInfo              Array                      [
                            {
                              "name": "inbound-rdp-rules",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkSecurity
                          Groups/aznewnsgbxmueijtaz47c/securityRules/inbound-rdp-rules",
                              "etag": "W/\"d214eb37-884a-49cc-b5c2-ada884fbf3f8\"",
                              "type": "Microsoft.Network/networkSecurityGroups/securityRules",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "description": "default nsg template for nic",
                                "protocol": "tcp",
                                "sourcePortRange": "*",
                                "destinationPortRange": "3389",
                                "sourceAddressPrefix": "*",
                                "destinationAddressPrefix": "VirtualNetwork",
                                "access": "Allow",
                                "priority": 100,
                                "direction": "Inbound",
                                "sourcePortRanges": [],
                                "destinationPortRanges": [],
                                "sourceAddressPrefixes": [],
                                "destinationAddressPrefixes": []
                              }
                            }
                          ]
                          nicInfo              Array                      [
                            {
                              "name": "ipconfig1",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkInterfac
                          es/aznewnicbxmueijtaz47c/ipConfigurations/ipconfig1",
                              "etag": "W/\"72b285f8-a277-4e41-a321-769bf9f5e53e\"",
                              "type": "Microsoft.Network/networkInterfaces/ipConfigurations",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "privateIPAddress": "10.0.0.4",
                                "privateIPAllocationMethod": "Dynamic",
                                "publicIPAddress": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/publicIPAdd
                          resses/aznewipbxmueijtaz47c"
                                },
                                "subnet": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetw
                          orks/aznewvnetbxmueijtaz47c/subnets/firstSubnet"
                                },
                                "primary": true,
                                "privateIPAddressVersion": "IPv4"
                              }
                            }
                          ]
                          vmInfo               Object                     {
                            "bootDiagnostics": {
                              "enabled": true,
                              "storageUri": "https://aznewstgbxmueijtaz47c.blob.core.windows.net/"
                            }
                          }
                          
DeploymentDebugLogLevel :
```

### <a name="vm-extensions"></a>10. Usage of VM extensions
|Property|Definition|
|---|---|
|Folder|[8-use-vm-extensions](./8-use-vm-extensions)|
|File|_azuredeploy.json_|

As we had seen earlier with the [bare-metal VM creation procedure](#resource-inter-dependency), we can leverage an option to use **"extensions" on-top of your VM to install/configure custom software, packages, services, etc** in a much more autonomous mode.

In this example, we will look

a. to create a **bare-metal VM** (like we had done in previous exercise)

b. and then **install IIS service** on-top of the VM, via _PowerShell_ utility

**Extract from the json file:**
```
/* resource definition */
...
{
    "type": "Microsoft.Compute/virtualMachines/extensions",
    "apiVersion": "2018-06-01",
    "name": "[concat(variables('vmName'),'/', 'InstallWebServer')]",
    "tags": "[parameters('resourceTags')]",
    "location": "[parameters('location')]",
    "dependsOn": [
        /* "[concat('Microsoft.Compute/virtualMachines/',variables('vmName'))]" */
        "[resourceId('Microsoft.Compute/virtualMachines',variables('vmName'))]"
    ],
    "properties": {
        "publisher": "Microsoft.Compute",
        "type": "CustomScriptExtension",
        "typeHandlerVersion": "1.7",
        "autoUpgradeMinorVersion":true,
        "settings": {
            "fileUris": [
                "https://raw.githubusercontent.com/Azure/azure-docs-json-samples/master/tutorial-vm-extension/installWebServer.ps1"
            ],
            "commandToExecute": "powershell.exe -ExecutionPolicy Unrestricted -File installWebServer.ps1"
        }
      }
  }
...
```

:pushpin: **Key Pointer(s):**

1. There are no-extra parameters to be defined in this simulation, except a new resource definiton of type: `"Microsoft.Compute/virtualMachines/extensions"`, which will **look to append the extensions on the created VM**

2. As **this resource is a direct inheritance from the VM**, it expects **the VM to be online & ready**, before instantiating the extension setup

3. The **"Powershell scripts"** need to be **hosted in an external server** (ex: _ftp server, or even in github_), and can be directly referenced within the _fileUris_ segment

4. The **appropriate command to be triggered** (i.e. installation of IIS service) can be invoked by using the _"commandToExecute_" segment

Now, lets proceed to run the command to **create a VM, and also have the IIS service pre-installed in it.**

**Command:**
```
PS C:\Users\nagarjun k\Documents\az-journey\arm\b-advance\8-use-vm-extensions> 
New-AzResourceGroupDeployment -Name "deployvmextensions" -ResourceGroupName "azure-lab-rg-01" \
-TemplateFile .\azuredeploy.json -storageAccountName "azdoc" -conditionParamsforStorage "new" -Verbose
```

**Output:**
```
VERBOSE: Performing the operation "Creating Deployment" on target "azure-lab-rg-01".
VERBOSE: 21:41:08 - Template is valid.
VERBOSE: 21:41:10 - Create template deployment 'deployvmextensions'
VERBOSE: 21:41:15 - Resource Microsoft.Network/publicIPAddresses 'aznewipbxmueijtaz47c' provisioning status is running
VERBOSE: 21:41:15 - Resource Microsoft.Network/networkSecurityGroups 'aznewnsgbxmueijtaz47c' provisioning status is running
VERBOSE: 21:41:15 - Resource Microsoft.Storage/storageAccounts 'azdoc' provisioning status is running
VERBOSE: 21:41:33 - Resource Microsoft.Network/networkInterfaces 'aznewnicbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 21:41:33 - Resource Microsoft.Network/virtualNetworks 'aznewvnetbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 21:41:33 - Resource Microsoft.Network/publicIPAddresses 'aznewipbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 21:41:33 - Resource Microsoft.Network/networkSecurityGroups 'aznewnsgbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 21:41:39 - Resource Microsoft.Storage/storageAccounts 'azdoc' provisioning status is succeeded
VERBOSE: 21:41:44 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is running
VERBOSE: 21:42:13 - Resource Microsoft.Compute/virtualMachines/extensions 'aznewvmbxmueijtaz47c/InstallWebServer' provisioning status is running
VERBOSE: 21:42:13 - Resource Microsoft.Compute/virtualMachines 'aznewvmbxmueijtaz47c' provisioning status is succeeded
VERBOSE: 21:46:10 - Resource Microsoft.Compute/virtualMachines/extensions 'aznewvmbxmueijtaz47c/InstallWebServer' provisioning status is succeeded

DeploymentName          : deployvmextensions
ResourceGroupName       : azure-lab-rg-01
ProvisioningState       : Succeeded
Timestamp               : 19-04-2020 16:16:12
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                          Type                       Value
                          ============================  =========================  ==========
                          uniqueName                    String                     aznew
                          storageAccountName            String                     azdoc
                          conditionParamsforStorage     String                     new
                          storagereplicationstrategy    String                     Standard_LRS
                          location                      String                     southindia
                          dnslabelprefix                String                     mynewapp
                          allAddrSpaces                 Object                     {
                            "addressPrefixes": [
                              {
                                "name": "firstvnetPrefix",
                                "addressPrefix": "10.0.0.0/16"
                              },
                              {
                                "name": "secondvnetPrefix",
                                "addressPrefix": "10.1.0.0/16"
                              }
                            ],
                            "subnets": [
                              {
                                "name": "firstSubnet",
                                "addressPrefix": "10.0.0.0/24"
                              },
                              {
                                "name": "secondSubnet",
                                "addressPrefix": "10.1.0.0/24"
                              }
                            ]
                          }
                          sizeofVM                      String                     Standard_DS1_v2
                          windowsImageSKU               String                     2019-Datacenter
                          computerName                  String                     mywinvm
                          adminAccount                  String                     azureadmin
                          adminPassword                 String                     Azurevm@123
                          resourceTags                  Object                     {
                            "environment": "simulation",
                            "lab-simulation": "arm-template-condition-based-resource-creation"
                          }

Outputs                 :
                          Name                 Type                       Value
                          ===================  =========================  ==========
                          publicIPAddress      String                     104.211.201.115
                          publicdnsSettings    Object                     {
                            "domainNameLabel": "mynewapp",
                            "fqdn": "mynewapp.southindia.cloudapp.azure.com"
                          }
                          vnetAddrSpace        Object                     {
                            "addressPrefixes": [
                              "10.0.0.0/16",
                              "10.1.0.0/16"
                            ]
                          }
                          vnetSubnet           Array                      [
                            {
                              "name": "firstSubnet",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/aznewvnetbxmueijtaz47c/subnets/firstSubnet",
                              "etag": "W/\"118e7991-29d0-4207-b65f-986fd9eac750\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.0.0.0/24",
                                "networkSecurityGroup": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkSecurityGroups/aznewnsgbxmueijtaz47c"
                                },
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            },
                            {
                              "name": "secondSubnet",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/aznewvnetbxmueijtaz47c/subnets/secondSubnet",
                              "etag": "W/\"118e7991-29d0-4207-b65f-986fd9eac750\"",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "addressPrefix": "10.1.0.0/24",
                                "delegations": [],
                                "privateEndpointNetworkPolicies": "Enabled",
                                "privateLinkServiceNetworkPolicies": "Enabled"
                              },
                              "type": "Microsoft.Network/virtualNetworks/subnets"
                            }
                          ]
                          nsgInfo              Array                      [
                            {
                              "name": "inbound-rdp-rules",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkSecurityGroups/aznewnsgbxmueijtaz47c/securityRules/inbound-rdp-rules",
                              "etag": "W/\"1b7be141-34ca-4333-8693-d97da97ad098\"",
                              "type": "Microsoft.Network/networkSecurityGroups/securityRules",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "description": "default nsg template for nic",
                                "protocol": "tcp",
                                "sourcePortRange": "*",
                                "destinationPortRange": "3389",
                                "sourceAddressPrefix": "*",
                                "destinationAddressPrefix": "VirtualNetwork",
                                "access": "Allow",
                                "priority": 100,
                                "direction": "Inbound",
                                "sourcePortRanges": [],
                                "destinationPortRanges": [],
                                "sourceAddressPrefixes": [],
                                "destinationAddressPrefixes": []
                              }
                            }
                          ]
                          nicInfo              Array                      [
                            {
                              "name": "ipconfig1",
                              "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkInterfaces/aznewnicbxmueijtaz47c/ipConfigurations/ipconfig1",
                              "etag": "W/\"fbd87497-4b86-4153-8253-c083680bce9c\"",
                              "type": "Microsoft.Network/networkInterfaces/ipConfigurations",
                              "properties": {
                                "provisioningState": "Succeeded",
                                "privateIPAddress": "10.0.0.4",
                                "privateIPAllocationMethod": "Dynamic",
                                "publicIPAddress": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/publicIPAddresses/aznewipbxmueijtaz47c"
                                },
                                "subnet": {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/virtualNetworks/aznewvnetbxmueijtaz47c/subnets/firstSubnet"
                                },
                                "primary": true,
                                "privateIPAddressVersion": "IPv4"
                              }
                            }
                          ]
                          vmInfo               Object                     {
                            "vmId": "569c0789-9a1b-4b91-80da-dc935cceb029",
                            "hardwareProfile": {
                              "vmSize": "Standard_DS1_v2"
                            },
                            "storageProfile": {
                              "imageReference": {
                                "publisher": "MicrosoftWindowsServer",
                                "offer": "WindowsServer",
                                "sku": "2019-Datacenter",
                                "version": "latest",
                                "exactVersion": "17763.1158.2004131759"
                              },
                              "osDisk": {
                                "osType": "Windows",
                                "name": "aznewvmbxmueijtaz47c_disk1_8fe98b8bdd5d4b3c82ed622778179261",
                                "createOption": "FromImage",
                                "caching": "ReadWrite",
                                "managedDisk": {
                                  "storageAccountType": "Premium_LRS",
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/AZURE-LAB-RG-01/providers/Microsoft.Compute/disks/aznewvmbxmueijtaz47c_disk1_8fe98b8bdd5d4b3c82ed622778179261"
                                },
                                "diskSizeGB": 127
                              },
                              "dataDisks": []
                            },
                            "osProfile": {
                              "computerName": "mywinvm",
                              "adminUsername": "azureadmin",
                              "windowsConfiguration": {
                                "provisionVMAgent": true,
                                "enableAutomaticUpdates": true
                              },
                              "secrets": [],
                              "allowExtensionOperations": true,
                              "requireGuestProvisionSignal": true
                            },
                            "networkProfile": {
                              "networkInterfaces": [
                                {
                                  "id": "/subscriptions/2f981ee7-6c60-4593-bc4b-82c9b050f722/resourceGroups/azure-lab-rg-01/providers/Microsoft.Network/networkInterfaces/aznewnicbxmueijtaz47c"
                                }
                              ]
                            },
                            "diagnosticsProfile": {
                              "bootDiagnostics": {
                                "enabled": true,
                                "storageUri": "https://azdoc.blob.core.windows.net/"
                              }
                            },
                            "provisioningState": "Succeeded"
                          }

DeploymentDebugLogLevel :
```
