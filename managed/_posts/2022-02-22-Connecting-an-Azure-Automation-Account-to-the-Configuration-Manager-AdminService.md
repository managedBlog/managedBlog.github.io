---
title: "Connecting an Azure Automation Account to the Configuration Manager AdminService"
excerpt: "This is it. In today’s post, I will cover how I access the AdminService for Automation from Azure."
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

_<small>I am really excited to share this post with you! This is the latest post in my series on automating endpoint management task, and we are finally going to talk about accessing AdminService from the cloud! [You can find the rest of the series here.](https://www.modernendpoint.com/tags/#microsoft-graph-api)</small>_

![Header](https://managedblog.github.io/managed/assets/images/22.02.22/00.Header.png){: .align-center}

Hello and welcome back! If you’ve been following along with my [current blog series](https://www.modernendpoint.com/tags/#microsoft-graph-api), you know that I have been working through the building blocks of automation with Microsoft Graph and the Configuration Manager AdminService. The entire series has been targeted at lowering the barriers to entry for systems administrators trying to leverage REST APIs in their automations. My earliest posts discussed the basics of REST APIs and then transitioned into accessing Microsoft Graph with PowerShell. More recently, I have been providing building blocks for managing automation tasks in Azure, but the one area I hadn’t covered was accessing the AdminService. 

This is it. In today’s post, I will cover how I access the AdminService for Automation from Azure. There are different ways to access the AdminService from the internet, but I will focus on using a hybrid worker group from an Azure Automation account. Before I jump into connecting your hybrid worker to Configuration Manager, I wanted to talk about the various methods that are available and why I settled on the method I did. Whenever I write a blog post, I want to make sure that I provide the best and most accurate information that I can. This post is no different. In the interest of full disclosure, I have spent a lot of time leading up to this post reviewing the different options. 


### Evaluating our options
----

To be somewhat objective, I decided to review the various options and weighed them against three criteria:

1.	_**Ease of set up**_ - How difficult is it to set up? Are there a lot of steps involved, and how difficult is it to accomplish? Does it need additional infrastructure? Certificates? Outside knowledge?

2.	_**Usability**_ - Can you make API calls directly to the service? Will it work with various automation entry points? Can it work from anywhere?

3.	_**Documentation and Support**_ How well documented is the solution? Is there supporting documentation available? Is the path generally supported, or is it treated as an edge use case?

I reviewed three different contenders. The first is accessing the AdminService through Cloud Management Gateway, the second is using Azure App Proxy to connect to AdminService, and finally, using a hybrid worker group.

#### Cloud Management Gateway

From looking at the documentation and the available features in Configuration Manager, it would appear that this would be the best and most natural fit. The official documentation for the AdminService is sparse, but much of the documentation that does exist covers connecting through the Cloud Management Gateway. There is also some great community content from [Sandy Zeng](https://msendpointmgr.com/2019/07/16/use-configmgr-administration-service-adminservice-over-internet/), [Nathan Ziehnert](https://z-nerd.com/blog/2019/12/05-working-with-adminservice-and-odata/), and [Adam Gross](https://www.asquaredozen.com/2019/02/12/the-system-center-configuration-manager-adminservice-guide/). 

Setting up access to the AdminService through CMG requires allowing access to the AdminService from the Internet in the CM console, setting up an app registration, and then getting an authorization token from the service when connecting. It works, but there’s a bit of overhead involved. The account used for accessing the AdminService must be synced from on-premise. The documentation seems to suggest that you must use interactive authentication or access the service from a trusted device. While I haven’t seen that explicitly stated, I also haven’t been able to prove that it’s not true. Connecting to the service requires getting an authorization token at the start of the process. Once you do get the CMG set up, it is undeniably fast.

Based on my criteria above, I would say that the CMG is the most difficult of the three options to set up. Once it is configured, it is great for use on tasks that will be performed interactively, but for other automation tasks it may be limited. When you take community content into account, it is fairly well documented. 

#### Azure App Proxy

Azure App Proxy allows us to expose the internal AdminService directly to the internet using the App Proxy service. It gets a passing mention in Microsoft’s official documentation, and there’s not a lot of community content out there. Configuring Azure App Proxy is well documented, just not specifically for accessing the AdminService.

If you’re reasonably familiar with Azure App Proxy, this connection can be set up in just a few minutes. Unfortunately, it has a lot of the same drawbacks from an ease-of-use perspective. You can pass credentials into a script and authenticate, but you need to be accessing the service from a trusted device. If you try to access the service directly from Azure, you receive a trust relationship error. 

#### Hybrid Worker Group

Azure Automation hybrid worker groups allow you to perform tasks against on-premise services. Agent based hybrid workers require a log analytics workspace and require quite a bit of time to configure. While this wasn’t a showstopper, it definitely made the hybrid runbook worker a lot more difficult to set up. [In my last post](https://www.modernendpoint.com/managed/Setting-up-a-Hybrid-Worker-on-an-Azure-Arc-Enabled-Server/), I covered setting up a Hybrid Worker Group on Azure Arc enabled servers. Onboarding an individual server is as easy as running a PowerShell script on the server and configure a Hybrid Worker Group in Azure Automation. 

I haven’t seen any content from Microsoft or the community that talks about accessing the AdminService using a hybrid worker. As far as I can tell, up until this post, it hasn’t been covered. However, services like Azure Automation and Arc are very thoroughly documented and already in production in many environments. While this specific use case hasn’t been documented, it is clearly supported because of the supportability of the underlying services. 

Most importantly, we have the question of usability. This option does require using an Azure Automation account and relies on the connection on the hybrid worker group. That is the biggest drawback, as any automation tasks must be run through Azure Automation. However, runbooks can be triggered through PowerShell and many other services (like Power Automate and Logic Apps) have built-in connectors as well.

Since the hybrid worker is running tasks on-premise, we don’t need to worry about exposing the AdminService directly to the internet. We don’t have to set up any additional app registrations, pass in credentials, or think about the trust relationship between Configuration Manager and the machine running the script. Using a hybrid worker also opens additional options for us – for example, if you wanted to get information about a user or device from Active Directory, you just need to make sure that the machine acting as a hybrid worker has the Remote Server Administration Tools installed.

One drawback I have found is that using a hybrid is a little bit slower. This is partly due to the nature of Azure Automation. Jobs are queued generally launch quickly, but then you have a few extra hops before the data is returned to the device. In most cases the difference is negligible. When planning automation workflows, you may want to take this into account and look for ways to optimize communication.


### On with the show – accessing the AdminService from a hybrid worker group
----

[My last post](https://www.modernendpoint.com/managed/Setting-up-a-Hybrid-Worker-on-an-Azure-Arc-Enabled-Server/) talked about how to configure an Azure Arc enabled server and a hybrid worker group in Azure Automation. If you haven’t read it yet, I highly recommend checking that post out first. The next steps in this post pick up where that left off. This post assumes that you have enabled have a hybrid worker group configured on an Azure Arc enabled server. Once the hybrid worker group is configured, the rest is fairly trivial and follows standard practices you should already be familiar with as a Configuration Manager administrator. 

If you are using a version of Configuration Manager older than version 2111, you will need to enable the AdminService in your CM environment. To do that, go to the Administration node and browse to Site Configuration > Sites. Click on Hierarchy Settings. On the general tab, check the box to, “Enable the Configuration Manager console to use the administration service.”

When using a hybrid worker on an Azure Arc enabled server, the service acts as that computer’s system account. The computer will need to have administrative rights in Configuration Manager to access the AdminService. We could Arc enable the CM server itself, but I prefer to use a separate server. By using a separate server, we can easily use that hybrid worker for multiple uses across the environment without an adverse effect on Configuration Manager. If we need to, we can use RBAC controls to further limit the access we grant to the hybrid worker. 

Inside of Configuration Manager open the Administration node. Click on Security and select Administrative Users.

![Select Administrative Users](https://managedblog.github.io/managed/assets/images/22.02.22/01.SelectAdministrativeUsers.png){: .align-center}

Click Add User or Group on the ribbon.

![Add User](https://managedblog.github.io/managed/assets/images/22.02.22/02.AddUser.png){: .align-center}

Click on the Browse button in the Add User or Group window. Click Object types and select the checkbox for Computers and click OK. Type in the server name for your hybrid worker, click Check Names and click OK.

![Search For Azure Arc Server](https://managedblog.github.io/managed/assets/images/22.02.22/03.SearchForAzureArcServer.png){: .align-center}

After adding the device, click Add to select security roles. For this example, I am assigning the server Full Administrator rights. Check the box for the security role you want to assign and click OK.

![Assign C M Role](https://managedblog.github.io/managed/assets/images/22.02.22/04.AssignCMRole.png){: .align-center}

Confirm all the settings are correct and click OK.

![Confirm Admin Role](https://managedblog.github.io/managed/assets/images/22.02.22/05.ConfirmAdminRole.png){: .align-center}
 

That’s it. You’re done. To recap, all we had to do to enable access to the Configuration Manager AdminService was:

1.	Create an Azure Automation account
2.	Enable our Azure subscription to use Azure Arc
3.	Download a PowerShell script to a local server
4.	Run the script as Administrator
5.	Create a Hybrid Worker group in our Azure Automation account
6.	Add the server to the Hybrid Worker Group
7.	Grant the server administrative rights to configuration manager

If you follow my posts to create an [Azure Automation account with PowerShell](https://www.modernendpoint.com/managed/Creating-an-Azure-Automation-Account/) and [set up a hybrid worker on an Azure Arc enabled server](https://www.modernendpoint.com/managed/Setting-up-a-Hybrid-Worker-on-an-Azure-Arc-Enabled-Server/) you can easily complete all of these steps in under 30 minutes without exposing the AdminService directly to the internet, setting up app registrations, or worrying about authentication to the service. You don’t have to worry about certificates to establish trust in the environment. It’s clean. It’s simple. It uses established Azure infrastructure tools. There is a small performance hit accessing the AdminService this way, and it doesn’t work well if you want to quickly run a single script from your machine, but in my opinion, it is unrivaled for running non-interactive automations from Azure.


### But does it work?
----

I have spent multiple blog posts building up to this moment. You could just trust me and believe this works, but you don’t have to take my word for it. To test my connection to the AdminService from an Azure Automation account, I created a script that will allow me to perform a simple REST API call from a runbook. 

This script will serve two purposes. First, it allows me to quickly test a REST API call to the AdminService using the Test Pane in the runbook. Second, as I will demonstrate in future posts, I will be able to leverage this same runbook for making single calls to the Admin Service from tools like Power Automate and Logic Apps. If this looks familiar, it is because it is based on the [Invoke-MsGraphCall function](https://github.com/managedBlog/Managed_Blog/blob/main/Microsoft%20Graph/Splatting/Invoke-MsGraphCall.ps1) that I wrote for making calls to Microsoft Graph. [The complete script is available from my Github repo.](https://github.com/managedBlog/Managed_Blog/blob/main/CM%20AdminService/Invoke-AdminServiceCall.ps1) _Note: This script is still being developed, any updates will be posted to GitHub. I expect that it may need a few minor tweaks prior to using it in low-code tools._

If we look at the script, there are a few things I want to call out. This script was optimized to run in an Azure Automation workbook. Because of the way Azure Automation handles input parameters, it doesn’t always handle hashtables well as parameters. To get around this limitation, I cast the `$BodyInput` variable as a string. This parameter is only needed for certain methods, including `POST`, `PUT`, and `PATCH`. 

If you are using one of those methods, make sure to test your string to make sure it properly gets converted. In the current script version, I would describe the string value as being JSON-like. It should be enclosed in curly brackets `{}`, use key-value pairs enclosed in double quotes, and the pairs should be comma separated.


````powershell
[cmdletBinding()]
param(
    [Parameter(Mandatory=$False)]
    [string]$URI,
    [Parameter(Mandatory=$False)]
    [string]$Method,
    [Parameter(Mandatory=$False)]
    [string]$BodyInput  
)

#Azure Automation cannot handle hashtable as a parameter, convert string pseudo-hash to hash table
$BodyInput = $BodyInput.replace("=",":")

#Convert the string to an actual hashtable
$BodyHash = @{}
$jsonobj = $BodyInput | ConvertFrom-Json
foreach($p in $jsonobj.psobject.properties){$BodyHash[$p.name] = $p.value}

````

The next section of the script is the function to make the REST API call. Inside of the function we create a splat that is nearly identical to the one used in the Invoke-MsGraphCall function I created in a [previous post](https://www.modernendpoint.com/managed/PowerShell-tips-for-accessing-Microsoft-Graph-in-PowerShell/). The biggest difference in this version is that I have added the parameter `UseDefaultCredentials = $True`. When we use Invoke-RestMethod to access the AdminService, we either need to specify credentials, or if accessing the API as the script invoker, specify `UseDefaultCredentials`.


````powershell
Function Invoke-AdminServiceCall {

    [cmdletBinding()]
    param(
        [Parameter(Mandatory=$True)]
        [string]$URI,
        [Parameter(Mandatory=$True)]
        [string]$Method,
        [Parameter(Mandatory=$False)]
        [string]$Body
    )

    #Create Splat hashtable
    $SplatParams = @{
        Headers     = @{
            "Content-Type"  = "application/json"
                    }
        Method = $Method
        URI = $URI
        UseDefaultCredentials = $True

    }

    #If method requires body, add body to splat
    If($Method -in ('PUT','PATCH','POST')){

        $SplatParams["Body"] = $Body

    }

    Write-Output $SplatParams

    #Return API call result to script
    $AsInvokeResult = Invoke-RestMethod @SplatParams #-UseDefaultCredentials

    #Return status code variable to script
    Return $AsInvokeResult

}
````

Finally, we need to convert our hashtable to a JSON object and call the function to make the REST call to the AdminService. The parameters needed for Invoke-AdminService call were set when we ran the script. It’s also worth noting that when you use PowerShell with Azure Automation, you need to use write-output to return the expected results.


````powershell
#Convert $BodyJson to hashtable
$BodyJson = $BodyHash | ConvertTo-Json

#Make REST API call to AdminService
$AdminServiceCall = Invoke-AdminServiceCall -URI $URI -Method $Method -Body $BodyJson

#Write results to output stream
Write-Output $AdminServiceCall
````

Open your Azure Automation account, click on runbooks, and create a new runbook. Name your runbook, set the type to PowerShell, and the runtime version to PowerShell 5.1. Click create. Copy and paste the complete script (included at the bottom of this post, or on [GitHub](https://github.com/managedBlog/Managed_Blog/blob/main/CM%20AdminService/Invoke-AdminServiceCall.ps1) to the script window. Click the Test Pane button to open the test pane.

First, we want to test the connection to the AdminService. We will do this by returning a single device from Configuration Manager. Set the URI to `https://Your-CmServer.FQDN.com/AdminService/wmi/SMS_R_System(12345678)`. Enter the FQDN of your CM server (or the server hosting the SMS provider role). Change the 8-digit ID to match the resource ID of a device in your CM environment.

For example, to return the device MME-WIN11-05 in my environment, I will use the following URI:
`https://mme-memcm01.mme-lab.com/AdminService/wmi/SMS_R_System(16777222)`

Set the method to `GET`. GET requests do not require a body, so we will leave the BodyInput parameter blank. Change the “Run on” slider to “Hybrid Worker,” and select the name of your hybrid worker group from the drop down. Click Start to start the test.

![Making Get Request](https://managedblog.github.io/managed/assets/images/22.02.22/06.MakingGetRequest.png){: .align-center}

The job will queue. If you watch the feed, you will see the status change to start, and once the job has completed the status will change to Completed. Here we can see that we were able to successfully return a device from the AdminService.

![Get Response Returned](https://managedblog.github.io/managed/assets/images/22.02.22/07.GetResponseReturned.png){: .align-center}

We have now proven that we can access the AdminService from Azure Automation, but what if we have a more complex request? Can we update a device in Configuration Manager from Azure? Let’s find out.

Inside of Configuration Manager, I have identified a device that doesn’t have a Primary User assigned. We will assign the primary user using the AdminService.

![Device With No Primary User](https://managedblog.github.io/managed/assets/images/22.02.22/08.DeviceWithNoPrimaryUser.png){: .align-center}

In this example, I want to update this device’s primary user. We can do this by making a `POST` call to `https:// Your-CmServer.FQDN.com /AdminService/wmi/SMS_UserMachineRelationship.CreateRelationship`. We know that POST requests require a JSON payload in the body of the request. This specific request requires the resource ID of a device, the user’s unique name from Configuration Manager, a SourceID value which specifies how the primary user was set, and a type ID. 

Ordinarily, I would pass these values in as a hashtable, but as mentioned above, Azure Automation doesn’t handle those well. In this case, I will pass in the following JSON-like string to generate the request body: `{ "UserAccountName"="MME-LAB\\Sean.Bulger","MachineResourceId"=16777222,"TypeId"= 1, "SourceId"=6 }`. Note that in this example, the numeric values are all integers. If you enclose these values as quotes, Configuration Manager will treat them as strings and will return an error. 

The values used in this example are:

URI: `https://mme-memcm01.mme-lab.com/AdminService/wmi/SMS_UserMachineRelationship.CreateRelationship`

Method: `POST`

BodyInput: `{ "UserAccountName"="MME-LAB\\Sean.Bulger","MachineResourceId"=16777222,"TypeId"= 1, "SourceId"=6 }`

Once again, we are setting the “Run on” slider to “Hybrid Worker” and selecting our hybrid worker group.

![Making P O S T Request](https://managedblog.github.io/managed/assets/images/22.02.22/09.MakingPOSTRequest.png){: .align-center}

After executing the runbook, we receive a response. This query didn’t return a response for a successful result (but if you check the AdminService log you will see the response code was 201, which is a successful run with an empty response).  If there had been an error, we would have received red text that would have included an error code.

![P O S T Response](https://managedblog.github.io/managed/assets/images/22.02.22/10.POSTResponse.png){: .align-center}

If we search for the device in Configuration Manager, we can see that the primary user has now been assigned.

![Primary User Updated](https://managedblog.github.io/managed/assets/images/22.02.22/11.PrimaryUserUpdated.png){: .align-center}
 

----

I have now demonstrated my preferred method of accessing the AdminService from Azure. I have been building up to this post for weeks, and I hope you find it both exciting and informative. The goal was to provide the best path for performing automation workloads, as this series is designed to assist endpoint administrators understand how to access and use the various REST APIs that are available to us. In future posts I will explore how to perform specific tasks with different automation tools in Azure. Thank you for following along so far, I hope you continue to visit for more great content!

----

As promised, here's the entire script that I broke down above:

````powershell

<#
Name: Invoke-AdminServiceCall.ps1
Author: Sean Bulger, twitter @managed_blog, http://managed.modernendpoint.com
Version: 0.1
.Synopsis
   This script will run a REST API call against the Configuration Manager Admin Service. It was create specifically for use in Azure Automation. Note: Azure Automation does not handle hashtable parameter input well, so the $Body parameter uses a JSON-like string instead of a hash table. 
   If running outside of Azure Automation, chang the parameter type to obj or hashtable based on your needs.
.DESCRIPTION
   Invoke-AdminServiceCall is a function built to call the Microsoft Configuration Manager AdminService and run any approved method.
#>

[cmdletBinding()]
param(
    [Parameter(Mandatory=$False)]
    [string]$URI,
    [Parameter(Mandatory=$False)]
    [string]$Method,
    [Parameter(Mandatory=$False)]
    [string]$BodyInput  
)


#Azure Automation cannot handle hashtable as a parameter, convert string pseudo-hash to hash table
$BodyInput = $BodyInput.replace("=",":")

#Convert the string to an actual hashtable
$BodyHash = @{}
$jsonobj = $BodyInput | ConvertFrom-Json
foreach($p in $jsonobj.psobject.properties){$BodyHash[$p.name] = $p.value}


#Create function to call the Admin Service
Function Invoke-AdminServiceCall {

    [cmdletBinding()]
    param(
        [Parameter(Mandatory=$True)]
        [string]$URI,
        [Parameter(Mandatory=$True)]
        [string]$Method,
        [Parameter(Mandatory=$False)]
        [string]$Body
    )


    #Create Splat hashtable
    $SplatParams = @{
        Headers     = @{
            "Content-Type"  = "application/json"
                    }
        Method = $Method
        URI = $URI
        UseDefaultCredentials = $True

    }

    #If method requires body, add body to splat
    If($Method -in ('PUT','PATCH','POST')){

        $SplatParams["Body"] = $Body

    }

    Write-Output $SplatParams

    #Return API call result to script
    $AsInvokeResult = Invoke-RestMethod @SplatParams #-UseDefaultCredentials

    #Return status code variable to script
    Return $AsInvokeResult

}

#Convert $BodyJson to hashtable
$BodyJson = $BodyHash | ConvertTo-Json

#Make REST API call to AdminService
$AdminServiceCall = Invoke-AdminServiceCall -URI $URI -Method $Method -Body $BodyJson

#Write results to output stream
Write-Output $AdminServiceCall

````
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
11. [Connecting an Azure Automation Account to the Configuration Manager AdminService](https://www.modernendpoint.com/managed/Connecting-an-Azure-Automation-Account-to-the-Configuration-Manager-AdminService)

