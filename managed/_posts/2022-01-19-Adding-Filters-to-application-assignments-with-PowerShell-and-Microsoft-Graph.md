---
title: "Adding Filters to application assignments with PowerShell and Microsoft Graph"
excerpt: "Over the course of the last two posts, we have been exploring how to create a PowerShell script to complete a task using Microsoft Graph. Now we need to put those elements together to create and run our script. "
tags:
  - Microsoft Endpoint Manager
  - Intune
  - Microsoft Graph API
  - AdminService
  - PowerShell
  - Challenges
---


_<small>Wait a minute - this post isn't the regularly scheduled post in my current series on automation in Microsoft Endpoint Manager with Microsoft Graph and the Admin Service - it's cool, but where do I find the rest of that series?  [You can find the entire series here.](https://www.modernendpoint.com/tags/#microsoft-graph-api)</small>_


I know you were expecting a post on the building blocks needed to automate tasks in Azure, but sometimes I get distracted by shiny objects. I promise that post is on its way, but this was too good to pass up!  The other day, [John Marcum (@MEM_MVP)]( https://twitter.com/MEM_MVP), asked a question on [Twitter](https://twitter.com/).

<div class="center">
<blockquote class="twitter-tweet">
<p lang="en" dir="ltr">Does anyone know of an easy way to add filters to all active <a href="https://twitter.com/hashtag/MSIntune?src=hash&amp;ref_src=twsrc%5Etfw">#MSIntune</a> app deployments? <a href="https://twitter.com/IntuneSuppTeam?ref_src=twsrc%5Etfw">@IntuneSuppTeam</a></p>&mdash; John Marcum (@MEM_MVP) <a href="https://twitter.com/MEM_MVP/status/1482039478206226443?ref_src=twsrc%5Etfw">January 14, 2022</a>
</blockquote><script async="" src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

Most people may have simply passed on without giving it a second thought. Not me. It has been running through my head for the last few days, and I decided to take it as a personal challenge. This is an excellent practical application of my [current blog series](https://www.modernendpoint.com/tags/#microsoft-graph-api), so I wanted to take on the challenge to see if I could do it.


_(Spoiler alert: Yes, I did it! That’s why you’re reading this post!)_

----

### The Challenge
----

To accomplish this task, we would need to be able to return a list of all the filters we have configured in Intune, a list of applications, and a list of application assignments. It seems like it should be relatively easy to update a simple value, but this one came with a few challenges:

1.	Only assigned apps can have filters applied, so we don’t want to return all devices
2.	Filters are tied to device types, so we only want to make sure we select the right filter for the applications we want to update
3.	There’s always an exception. The original ask was to apply a filter to all applications. I wanted to make sure we could select only the applications we wanted to update.
4.	It took me a while to realize this, but the body needed to update the assignments is complex. We would need to be able to dynamically create it based on the application type.
5.	We will need to pass in a parameter that sets the filter to either `include` or `exclude` devices


### The Solution
----

Ultimately, updating a single application assignment requires four total API calls:

1.	A `GET` request that returns all filters created in Intune
2.	A `GET` request that returns all applications with assignments
3.	A `GET` request that returns all assignments on an application
4.	A `POST` request that delivers a payload to update application assignments

The Microsoft Authentication Library module (MSAL.ps) is required for the script to run. When I am running PowerShell scripts against Microsoft Graph, I always use some form of authentication with MSAL. In this case, I am using a Client Secret from an Azure App Registration. [You can read more about using the MSAL library in PowerShell here.](https://www.modernendpoint.com/managed/connecting-to-microsoft-graph-with-powershell/) 

The only required parameter to run the script is the filter mode. When running the script, use the `-FilterMode` parameter to set a value of either `include` or `exclude`.


![Enter Filter Mode Parameter](https://managedblog.github.io/managed/assets/images/22.01.19/01.EnterFilterModeParameter.png){: .align-center}


To limit the number of required parameters and allow for an administrator to target only specific applications, I elected to pipe the results of the first two `Get` requests to `Out-GridView`. For those of you who aren’t familiar, `Out-GridView` displays an interactive box that allows you to select one or multiple objects from a list. Using the `-PassThru` parameter will pass the selected object back to a variable in your script.

I did elect not to prompt the user for input when selecting assignments on a specific application. The original ask was to be able to update all applications, so I didn’t want to prompt a user dozens of times to select specific assignments on different applications.

The heart of the script is the `Invoke-MSGraphCall` function that [I created in my earlier blog post on working with JSON and splatting in PowerShell](https://www.modernendpoint.com/managed/PowerShell-tips-for-accessing-Microsoft-Graph-in-PowerShell/). Each time I call the function, I pass in the specific parameters needed for that API call.

The full text of this function is:

````powershell
#Function to make Microsoft Graph API calls
Function Invoke-MsGraphCall {

    [cmdletBinding()]
    param(
        [Parameter(Mandatory=$True)]
        [string]$AccessToken,
        [Parameter(Mandatory=$True)]
        [string]$URI,
        [Parameter(Mandatory=$True)]
        [string]$Method,
        [Parameter(Mandatory=$False)]
        [string]$Body
    )

    #Create Splat hashtable
    $graphSplatParams = @{
        Headers     = @{
            "Content-Type"  = "application/json"
            "Authorization" = "Bearer $($AccessToken)"
        }
        Method = $Method
        URI = $URI
        ErrorAction = "SilentlyContinue"
        StatusCodeVariable = "scv"
    }

    #If method requires body, add body to splat
    If($Method -in ('PUT','PATCH','POST')){

        $graphSplatParams["Body"] = $Body

    }

    #Return API call result to script
    $MSGraphResult = Invoke-RestMethod @graphSplatParams

    #Return status code variable to script
    Return $SCV, $MSGraphResult

}
````

The first API call we make is to return the assignment filters we have configured in Endpoint Manager. We must use a filter type that matches the application whose assignments we wish to filter, so be careful to select the right filter for your device type.

The code to complete this task is as follows:

````powershell
#Return a list of all filters
$Method = "GET"
$URI = "https://graph.microsoft.com/beta/deviceManagement/assignmentFilters"

$MSGraphCall = Invoke-MsGraphCall -AccessToken $AccessToken -URI $URI -Method $Method -Body $Body

#Parse returned object to set Filters variable
$Filters = $MSGraphCall.value

#Show user grid view to select filter
Write-Host "Please select a filter to continue..." -ForegroundColor Yellow
$FilterId = $Filters | Out-GridView -PassThru | Select-Object -ExpandProperty id
````

This will return the following output:


![Select A Filter](https://managedblog.github.io/managed/assets/images/22.01.19/02.SelectAFilter.png){: .align-center}


Select a filter and click OK to continue. This will set the value of `$FilterId` to the selected filter.

Next, we need to return a list of all applications with assignments. I accomplished that with this code:

````powershell
#Get a list of all assigned applications. Script will only update applications with existing assignments
$Method = "GET"
$URI = "https://graph.microsoft.com/beta/deviceAppManagement/mobileApps?`$filter=isAssigned eq true"

$MSGraphCall = Invoke-MsGraphCall -AccessToken $AccessToken -URI $URI -Method $Method -Body $Body

#Pipe a list of all assigned apps to Out-GridView. User may select one or many apps.
$AssignedApps = $MSGraphCall[1].value

Write-Host "Please select the apps you would like to update to continue. ***NOTE: App types must match application type used in filter!" -ForegroundColor Yellow
$AppsToUpdate = $AssignedApps | Select-Object displayName,`@odata.type,id | Out-GridView -PassThru

````

This script block will return the following box. This supports multiple applications. Select one or more applications and click OK.


![Select Applications To Update](https://managedblog.github.io/managed/assets/images/22.01.19/03.SelectApplicationsToUpdate.png){: .align-center}


The rest of the script will run without any user input. Because of the `ForEach` loop, we need to clear the variables in case it had been previously run. Not all applications have the same properties and leaving the values from a previous run may cause the script to fail. We then iterate through each application in the list and return their assignments. 

````powershell
ForEach($ATU in $AppsToUpdate){

    #Null values used to create hashtable. Apps of different odata types may not have all properties, and failing to null variables before running may cause script to fail!
    $Target = $null
    $Settings = $null
    $TargetHash = $null
    $SettingsHash = $null

    #Get App Assignments
    $AppID = $ATU.id

    $Method = "GET"
    $URI = "https://graph.microsoft.com/beta/deviceAppManagement/MobileApps/$AppID/assignments"

    $MSGraphCall = Invoke-MsGraphCall -AccessToken $AccessToken -URI $URI -Method $Method -Body $Body

    #Process each app assignment on applications to apply filter. Script is currently built to apply same filter to ALL assignments.
    $AssignmentIDs = $MSGraphCall.value
````

The next block of code will run through each individual assignment on a device. This will parse the results for the `Target` and `Settings` of each assignment. The target contains information about the group the assignment is being applied to and contains the information about any filters being applied. We are updated the filter values on the target. Settings contains specific information about the assignment. In this version of the script, we aren’t changing any settings. Both the target and the settings have to included in the JSON payload. Both the target and settings could vary by application type, so we need to parse the individual objects to make sure we include the correct properties in the correct format. We need to convert the hash table to JSON, and the JSON needs to have sufficient depth to account for different application and assignment types.

````powershell
ForEach($AID in $AssignmentIDs){}

    #Set values to create hashtable. Settings will use all existing values, target will update app assignment with filter information. All other values remain the same
    $Target = $AID.target
    $Settings = $AID.settings

    $Target.deviceAndAppManagementAssignmentFilterId = $FilterId
    $Target.deviceAndAppManagementAssignmentFilterType = $FilterMode

    #Create target hashtable
    $TargetHash = [ordered]@{}
    $Target.psobject.Properties | ForEach-Object { $TargetHash[$_.Name] = $_.Value }

    #Create hashtable for API call body
    $Hash = [ordered]@{ 

        "@odata.type" = "#microsoft.graph.mobileAppAssignment"
        "intent" = "Required"
        "target" = $TargetHash
        
    } 

    #If settings were returned with assignments, create settings hash and add it to hash
    If($Settings){

        $SettingsHash = [ordered]@{}
        $Settings.psobject.Properties | ForEach-Object { $SettingsHash[$_.Name] = $_.Value }
        $Hash.Insert(2, 'settings', $SettingsHash)

    }

    #Note: Different app types have different JSON depths. I haven't found any that go deeper than 4, but this is a potential point of failure.
    $MobAppAssignments = @{ "mobileAppAssignments" = @($Hash) } | ConvertTo-Json -Depth 5
````

Finally, we make a `POST` request to update the application assignments. We pass in the JSON payload in the `-body` parameter.

````powershell
    #Update the management name of the device at the URI for each device ID
    $URI = "https://graph.microsoft.com/beta/deviceAppManagement/mobileApps/$AppId/assign"
    $Body = $MobAppAssignments
    $Method = "POST"

    #Call Invoke-MsGraphCall to assign filters to applications
    $MSGraphCall = Invoke-MsGraphCall -AccessToken $AccessToken -URI $URI -Method $Method -Body $Body
````

If we check one of the applications we just updated, we can see that the filter has been applied.


![Application Assignment Filter Added](https://managedblog.github.io/managed/assets/images/22.01.19/04.ApplicationAssignmentFilterAdded.png){: .align-center}


[The full script can be found in my GitHub repository.](https://github.com/managedBlog/Managed_Blog/blob/main/Microsoft%20Graph/Assignment%20Filters/Add-AssignmentFilters.ps1)

_____

Sure, this post wasn't intended as part of my series on automation with Microsoft Graph, but it's a natural fit! I'm going to include it in the list because it deserves to be here!

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
12. [Running an Azure Automation runbook to update MEMCM Primary User](https://www.modernendpoint.com/managed/Running-an-Azure-Automation-runbook-to-update-MEMCM-Primary-User)
13. [Using Power Automate to update MEMCM Device based on Intune User](https://www.modernendpoint.com/managed/Using-Power-Automate-to-update-MEMCM-Device-based-on-Intune-User)