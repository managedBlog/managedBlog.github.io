---
title: "Creating an Azure Automation Account"
excerpt: "There are a lot of different tools that can be used to automate tasks in Azure, but perhaps the most flexible is an Azure Automation account."
tags:
  - Microsoft Endpoint Manager
  - Intune
  - Microsoft Graph API
  - AdminService
  - PowerShell
  - Azure
  - Azure Automation
  - Managed Identity
---

_<small>This is the latest installment in my series on endpoint automation with Microsoft Graph and the AdminService. I have been providing the building blocks to automate endpoint management tasks in your environment. [You can find the rest of the series here.](https://www.modernendpoint.com/tags/#microsoft-graph-api)</small>_



Welcome back! This is the latest post in my series on Endpoint Automation with Microsoft Graph and the Admin Service! To this point in the series, we have talked about the basics of Rest APIs and Microsoft Graph. In my last post I covered Azure Key Vault and began laying the foundation for automating tasks in Microsoft Azure. This post continues that theme. There are a lot of different tools that can be used to automate tasks in Azure, but perhaps the most flexible is an Azure Automation account.

[Azure Automation](https://docs.microsoft.com/en-us/azure/automation/overview) is a service that allows for automation and orchestration of your environment from the Azure cloud. Tasks can be automated across your Azure services and on-premise. By leveraging a Hybrid Runbook Worker, we can run scripts against our on-premise Active Directory and on-premise servers. This provides a seamless way to leverage your existing scripts to automate tasks across your entire infrastructure. We can trigger tasks directly from our Automation Account, but we can also integrate with other tools. Connectors in Power Automate and Logic Apps connect to Azure Automation and provide a low-code solution to quickly deploy automation workloads.

In this post, we are going to create an Automation Account. Once we have created the Automation Account, we will set up a managed identity, and then create an access policy in Azure Key Vault to allow the managed identity to retrieve stored keys and secrets. This post assumes that you have already created an Azure Key Vault and stored a secret as I [described in my last post](https://www.modernendpoint.com/managed/Working-with-Azure-Key-Vault-in-PowerShell/).

----
### Setting up an Automation Account using the Azure Portal
----

Sign into the [Azure Portal](https://portal.azure.com/#home). Click on Resource Groups and click on the name of your resource group to open it. Click “+ Create” to create a new resource.
 
![Create New Resource](https://managedblog.github.io/managed/assets/images/22.02.07/01.CreateNewResource.png){: .align-center}

In the search box type “Automation” and press enter. Click on “Create” and select “Automation” to create a new Automation account.

![Create An Automation Account](https://managedblog.github.io/managed/assets/images/22.02.07/02.CreateAnAutomationAccount.png){: .align-center}

Name your Automation account and select a region, then click “Review + Create.” 

![Resource Group Settings](https://managedblog.github.io/managed/assets/images/22.02.07/03.ResourceGroupSettings.png){: .align-center}

On the confirmation page, click “Create” to create the automation account. Once the account has been created, click “Go to resource” to open your Automation Account. 

![New Automation Account](https://managedblog.github.io/managed/assets/images/22.02.07/04.NewAutomationAccount.png){: .align-center}

The first thing we need to do with our Automation Account is to set up a Managed Identity. A Managed Identity is an Azure managed service principal that doesn’t require you to provision or manage any secrets. It can be used across various Azure Resources, but we will primarily use it to access secrets stored in Azure Key Vault. In the past, Azure Automation used a RunAs account to access shared resources, but those accounts required an administrator to manage secrets. While they are still available, managed identities are now the preferred option for automation workloads.

To create a managed Identity, click “Identity” under Account Settings.

![Click On Identity](https://managedblog.github.io/managed/assets/images/22.02.07/05.ClickOnIdentity.png){: .align-center}
 
Move the slider to “On” to turn on the system-assigned managed identity. A system-assigned managed identity is tied to a specific resource and is available for the life of that resource. Only the resource that created an identity can use it. A user-assigned managed identity is created as a standalone resource that can be assigned to other resources. For the purposes of this blog post, a system-assigned managed identity is preferred.

After we enable the managed identity, we need to assign a role to it. Click “Azure Role Assignments.”

![Enable Managed Identity](https://managedblog.github.io/managed/assets/images/22.02.07/06.EnableManagedIdentity.png){: .align-center}

Select the correct subscription then click “Add Role Assignment.”

![Add Role Assignment](https://managedblog.github.io/managed/assets/images/22.02.07/07.AddRoleAssignment.png){: .align-center}

Set the scope to “Subscription” and the role to “Contributor.” This will allow the managed identity to act as a contributor with broad access, but limited control over resources in the subscription.

![Add Assignment Permissions](https://managedblog.github.io/managed/assets/images/22.02.07/08.AddAssignmentPermissions.png){: .align-center}

The managed identity needs to have access to return keys from our Key Vault. Copy the Object Id from the managed Identity page. Open your Azure Key Vault from the Resource Group. Inside of the Key Vault, click on “Access Policies” under Settings.

![Azure Key Vault Menu](https://managedblog.github.io/managed/assets/images/22.02.07/09.AzureKeyVaultMenu.png){: .align-center}

Click ”+ Add Access Policy” to create a new Access policy. On the Add Access Policy page, set the Key Permissions and Secret Permissions to “Get,” click “none selected” next to Select principal. Search for the Object Id of your managed Identity, click the account, and then click “Select.” Click “Add” to save the Access Policy.

![Add Access Policy](https://managedblog.github.io/managed/assets/images/22.02.07/10.AddAccessPolicy.png){: .align-center}

Under Application Access Policies, we can see that the access policy was created. Click “Save” on the ribbon to save changes.

![Access Policy Added](https://managedblog.github.io/managed/assets/images/22.02.07/11.AccessPolicyAdded.png){: .align-center}


Later in the post we will create a simple runbook that will test our managed identity and its ability to return a secret from Azure Key Vault, but first I will demonstrate how to create our Automation Account through PowerShell.

----
### I got a fever, and the only cure is more PowerShell!
----


We can also create an Automation Account through PowerShell. I have added [a new script to my GitHub](https://github.com/managedBlog/Managed_Blog/blob/main/Azure%20Administration/Create-AutomationAccount.ps1) that fully automates the process. This is built using the same basic format that I used in my [Create-NewKeyVault](https://github.com/managedBlog/Managed_Blog/blob/main/Azure%20Administration/Create-NewKeyVault.ps1) script. _(Expect to see this functionality incorporated in a new module or community tool at some point in the future!)_ For the sake of brevity, I am only going to cover the steps that are required to create the automation account in this post. If you want to find more information on how to create a key vault, please reference [my previous blog post](https://www.modernendpoint.com/managed/Working-with-Azure-Key-Vault-in-PowerShell/). This script requires the Az PowerShell module. If you are running the complete script from my GitHub you will need to enter a name for your Azure Automation Account. Other parameters are optional, but the script will also prompt you for any additional values that it needs.
 
These examples can be run independently of the provided script, but please note you will need to provide your own values for the variables. 
 
First, create the new Automation Account. We will use the `New-AzAutomationAccount` cmdlet to complete this task. 

````powershell 
#Declare variables required to run outside of main script 
$AzAutomationAcctName = "MME-Automation-Demo" 
$Region = "eastus" 
$ResourceGroupName = "MME-AKVPOST-RG" 
  
#Create new Automation Account 
$NewAutoAccount = New-AzAutomationAccount -Name $AzAutomationAcctName -Location $Region -ResourceGroupName $ResourceGroupName 
$NewAutoAccount 
```` 
 
We can see that the Automation Account was created: 

![P S Automation Account Created](https://managedblog.github.io/managed/assets/images/22.02.07/12.PSAutomationAccountCreated.png){: .align-center}
 
Next, we need to enable the managed identity. We do this using the `Set-AzAutomationAccount` cmdlet with the `-AssignSystemIdentity` parameter 
 
 ````powershell
#Enable system Identity 
$AssignId = Set-AzAutomationAccount -ResourceGroupName $resourceGroupName -Name $AzAutomationAcctName -AssignSystemIdentity 
$AssignId 
````

`Set-AzAutomationAccount` returns the updated object. We can see that the `Identity` property has been populated with a new object of type ` Microsoft.Azure.Management.Automation.Models.Identity`. If we return the `Identity` property of the resource, we can see that the object ID for the managed identity is stored as `$AssignId.identity.PrincipalId`. 

![Managed Identity Created](https://managedblog.github.io/managed/assets/images/22.02.07/13.ManagedIdentityCreated.png){: .align-center}
 
Next, we create our role assignment. We need to pass in the `PrincipalId` of the Managed Identity for the `ObjectId` parameter and the subscription ID for our subscription as part of the `Scope` parameter.  
 
````powershell
$AutIdentityId = $AssignId.identity.PrincipalId   
$SubId = "93b5e2d1-1c25-4902-a322-6a7ea61265c6" 
 
#Assign Role assignment and grant GET permissions to get keys from Azure Key Vault 
New-AzRoleAssignment -ObjectId $AutIdentityId -Scope "/subscriptions/$subId" -RoleDefinitionName "Contributor" 
````

The resulting output of the `New-AzRoleAssignment` cmdlet shows that the role was assigned contributor access to the subscription. 
 
![Contributor Role Assigned](https://managedblog.github.io/managed/assets/images/22.02.07/14.ContributorRoleAssigned.png){: .align-center}
 
Finally, we need to assign an access policy to be able to retrieve keys from Azure Key Vault. Because we will primarily use our managed identity for automation workloads, it only needs to be able to return keys and secrets. We will assign `GET` permissions for both secrets and keys. 
 
````powershell
Set-AzKeyVaultAccessPolicy -VaultName $AzKeyVaultName -ObjectId $AutIdentityId -PermissionsToKeys GET -PermissionsToSecrets GET 
````

----
### But will it work?
---- 


We are ready to test out Automation Account by running a simple runbook. I will cover runbooks more in depth in a later post, but we do want to make sure that we can use our managed identity to return a key from Azure Key Vault. Access your Automation Account by opening your resource group and clicking on the Automation Account in the list of resources.

![A K V Resource Group](https://managedblog.github.io/managed/assets/images/22.02.07/15.AKVResourceGroup.png){: .align-center}

Click on “Runbooks” under Process Automation. 

![Click On Runbooks](https://managedblog.github.io/managed/assets/images/22.02.07/16.ClickOnRunbooks.png){: .align-center}

Click “+ Create a Runbook” to create a new runbook.

![Create A Runbook](https://managedblog.github.io/managed/assets/images/22.02.07/17.CreateARunbook.png){: .align-center}

We are going to create PowerShell runbook. Name your runbook, select a version, and then click “Create.” 
 
![Runbook Details](https://managedblog.github.io/managed/assets/images/22.02.07/18.RUnbookDetails.png){: .align-center}

When you first create a runbook, you will be taken directly to a script editor. Copy the following PowerShell code into the editor and click “Save”

````powershell
#Connect to Azure Key Vault using managed Identity from within Azure Automation
Connect-AzAccount -Identity

#Return secret from Azure Key Vault
$ConnectionSecrets = Get-AZKeyVaultSecret -VaultName "MME-AKV-Blog" -Name "MME-ML-KEY" -AsPlainText

Write-Output $ConnectionSecrets
````

We can test the script by clicking on the “Test Pane” icon on the page ribbon.

![Save Power Shell Runbook](https://managedblog.github.io/managed/assets/images/22.02.07/19.SavePowerShellRunbook.png){: .align-center}

On the test pane, click Start. 

![Runbook Test Pane](https://managedblog.github.io/managed/assets/images/22.02.07/20.RunbookTestPane.png){: .align-center}

Once the script run completes, we can see that the multiline secret we had previously stored in Azure Key Vault is available to us. 

![Runbook Test Completed](https://managedblog.github.io/managed/assets/images/22.02.07/21.RunbookTestCompleted.png){: .align-center}

Our Automation Account was successfully configured. We have validated that the managed identity is available and able to access keys stored in the key vault. While there are ways to automate tasks without the Automation Account, it will open a lot of options to us. In future blog posts we will talk about how to use our Automation Account with tools like log analytics, a hybrid runbook worker, and logic apps to automate management tasks across our entire endpoint management landscape.

______

Follow the full series below:

1. [Everything I wanted to know about APIs but was afraid to ask](https://www.modernendpoint.com/managed/everything-i-wanted-to-know-about-apis-but-was-afraid-to-ask/)
2. [Connecting to Microsoft Graph with PowerShell](https://www.modernendpoint.com/managed/connecting-to-microsoft-graph-with-powershell/)
3. [Troubleshooting Microsoft Graph App Registration Errors](https://www.modernendpoint.com/managed/troubleshooting-microsoft-graph-app-registration-errors/)
4. [Defining a script and finding the Microsoft Graph Calls](https://www.modernendpoint.com/managed/Defining-a-script-and-finding-the-Microsoft-Graph-Queries/)
5. [Splatting with Invoke-RestMethod in PowerShell](https://www.modernendpoint.com/managed/PowerShell-tips-for-accessing-Microsoft-Graph-in-PowerShell/)
6. [Updating Device Management name in PowerShell with Microsoft Graph](https://www.modernendpoint.com/managed/Updating-device-management-name-in-PowerShell-with-Microsoft-Graph/)
7. [Surprise Post:Adding Filters to application assignments with PowerShell and Microsoft Graph](https://www.modernendpoint.com/managed/Adding-Filters-to-application-assignments-with-PowerShell-and-Microsoft-Graph/)
8. [Working with Azure Key Vault in PowerShell](https://www.modernendpoint.com/managed/Working-with-Azure-Key-Vault-in-PowerShell/)
9. [Creating an Azure Automation Account](https://www.modernendpoint.com/managed/Creating-an-Azure-Automation-Account/)
10. [Setting up a Hybrid Worker on an Azure Arc Enabled Server](https://www.modernendpoint.com/managed/Setting-up-a-Hybrid-Worker-on-an-Azure-Arc-Enabled-Server/)
