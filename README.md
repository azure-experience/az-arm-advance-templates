# az-arm-advance-templates
A detailed run on few of the advance topics of ARM

## Index for different user-cases
### [1. Brief Introduction _(Before you start)_](#before-we-start)
### [2. Resource creation](#new-resource-creation)
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

