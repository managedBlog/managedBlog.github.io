---
title: "Updating device management names in PowerShell with Microsoft Graph"
excerpt: "Over the course of the last two posts, we have been exploring how to create a PowerShell script to complete a task using Microsoft Graph. Now we need to put those elements together to create and run our script. "
tags:
  - Microsoft Endpoint Manager
  - Intune
  - Microsoft Graph API
  - AdminService
  - PowerShell
---


_<small>Welcome back! This post is a continuation on my series about endpoint management automation using Microsoft Graph and (eventually) the CM AdminService. This series is design to provide the building blocks for systems administrators to understand how to work with REST APIs in PowerShell.  [You can find the rest of the series here.](https://www.modernendpoint.com/tags/#microsoft-graph-api)</small>_


| ![header](https://managedblog.github.io/managed/assets/images/22.01.04/Depositphotos_42826349_XL.png){: .align-center} |

Over the course of the last two posts, we have been exploring how to create a PowerShell script to complete a task using Microsoft Graph. [In the earlier post](https://www.modernendpoint.com/managed/Defining-a-script-and-finding-the-Microsoft-Graph-Queries/), I talked about how to define a script and identify the API calls you would need to make. [In my last post](https://www.modernendpoint.com/managed/PowerShell-tips-for-accessing-Microsoft-Graph-in-PowerShell/), I walked through how to make Microsoft Graph calls in PowerShell and [created a function](https://github.com/managedBlog/Managed_Blog/blob/main/Microsoft%20Graph/Splatting/Invoke-MsGraphCall.ps1) that can be used to make any API call to Microsoft Graph. Now we need to put those elements together to create and run our script. 

----

### The Elements
----

We have already defined that our script should change a device’s management name based on the user’s Office, department, city, and the device’s serial number. To limit the scope of the script, I elected to run it on a per-user basis. 
By passing in a User Principal Name, we will return a user object and select several attributes. After returning the user object, we will also return their devices. We will use the user and device attributes to build a string that we will use for the management name. Once we have generated the management name, we will PATCH the managed device in Azure. In a full production script, we would also add logging and error handling. For ease of demonstration, I will keep this script simple and only focus on the functional script elements. 
We will use the Invoke-MsGraphCall function to make each call to Microsoft Graph. The rest of the script elements will be completed with standard PowerShell functionality. 
Before creating a script, let’s create a pseudocode outline of the script:

1.	Define Parameters – For our script to run, we need to pass in a UserPrincipalName as a parameter. If you're new to PowerShell, parameters are defined by adding a `param()` statement at the start of a script or function.

2.	Authenticate and get an access token – When we make calls to Microsoft Graph, we need to pass an access token in the headers of our request. As long as the underlying service principal has the correct rights, we can use the same access token for all of our API calls. [You can read more about that here.](https://www.modernendpoint.com/managed/connecting-to-microsoft-graph-with-powershell/)

3.	Declare functions – In this script, we are using a single function that will be used to make our calls to Microsoft Graph. [I created the Invoke-MsGraphCall function as part of my last blog post](https://www.modernendpoint.com/managed/PowerShell-tips-for-accessing-Microsoft-Graph-in-PowerShell/). We will use that function as the heart of this script.

4.	Return our Azure AD user object – we can return a specific user object by making a `GET` request to `https://graph.microsoft.com/v1.0/users/[UserPrincipalName]`. We will limit the number of attributes returned by using a select query: `?$select=displayName,officeLocation,city,usageLocation,department,mailNickname`

5.	Define the user-based values that will be used in the management name – using the full city, office, or department names for a user will create a longer management name. Spaces or special characters in attributes might also create unintended consequences in our script. We will use hashtables to define specific values to use in the management name.

6.	Return the user’s devices – We can return a specific user’s devices by making a `GET` request to `https://graph.microsoft.com/v1.0/deviceManagement/managedDevices`. This will return all devices, so we will append the request with a filter, `?$filter=userPrincipalName eq '$userPrincipalName'`. We want to limit the attributes returned, so we will also use a select query. Use an ampersand `&` after the filter query to append the select statement, `&$select=id,deviceName,serialNumber,model`.

7.	Iterate through each device returned and complete the following:

    1.	Return the `deviceId` to the `$DevId` variable 
    2.	Use the serial number as part of the management name string. If there is no serial number or the device is a virtual machine, use the device name instead.
    3.	Create the management name string
    4.	Make a `PATCH` request to `https://graph.microsoft.com/beta/deviceManagement/managedDevices/$DevId` to update the device’s management name

### The Script
----

We have defined the elements we need for our script and built an outline. We can now write a script to handle each of the defined tasks. I will break down each element below. [You can find the completed script in my GitHub Repo.](https://github.com/managedBlog/Managed_Blog/blob/main/Microsoft%20Graph/Update-ManagmentName.ps1)

First, we need to define the parameters for our script. We are updating all of the devices for a specific user, so we need to pass in information about the user for the script to run. I have defined one parameter, `UserPrincipalName`, and made it required. This should appear at the top of your script, immediately after any definitions.

````powershell
param(

    [Parameter(Mandatory=$True)]
    [string]$UserPrincipalName

)
````

We will need an access token to pass to Microsoft Graph for our script to run successfully. I prefer to get the access token at the start of my script. Traditionally you declare any variables at the start of a script, and I treat the access token as any other variable.

````powershell
#Use a client secret to authenticate to Microsoft Graph using MSAL
$authparams = @{
    ClientId    = '[Your Client Id]'
    TenantId    = 'YourDomain.com'
    ClientSecret = ('[Your Client Secret]' | ConvertTo-SecureString -AsPlainText -Force )
}

$auth = Get-MsalToken @authParams

#Set Access token variable for use when making API calls
$AccessToken = $Auth.AccessToken

````

After we have declared variables, we can add functions to the script. For this step, I am simply pasting in the Invoke-MsGraphCall function. 

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

Next, we need to return the user object. We are going to start by creating the parameters. The method is `GET`, but the URI will need to be constructed. We defined above that we need to return a specific user by User Principal Name, and then select specific fields to create the management name. 

We can return the user object by making a `GET` request to `https://graph.microsoft.com/v1.0/users/$UserPrincipalName`. We will select specific attributes by appending a select query, `?$select=displayName,officeLocation,city,usageLocation,department,mailNickname`.

Finally, we make the graph call and return the results to the `$MSGraphCall` variable.

````powershell
#Return user object
#Create parameters to use in Invoke-MSGraphCall
$UserFilter = '?$select=displayName,officeLocation,city,usageLocation,department,mailNickname'
$UserRoot = "https://graph.microsoft.com/v1.0/users"
$URI = "$UserRoot/$UserPrincipalName/$UserFilter"
$Method = "GET"

#Call Invoke-MsGraphCall
$MSGraphCall = Invoke-MsGraphCall -AccessToken $AccessToken -URI $URI -Method $Method -Body $Body
````

If we run the script up to this point and return the `$MSGraphCall` variable, we see the status code variable and user object have been returned as expected.

![User Object Returned](https://managedblog.github.io/managed/assets/images/22.01.04/01.UserObjectReturned.png){: .align-center}
----

Now that we have returned the user object, we can use the resulting object to create the various elements we will use in the management name. `$MSGraphCall` is an array. The first item in the array is the status code variable, and the second item is our resulting use object. Set a new variable to select the second item in the array. Arrays start at position 0, so we need to select the object in position 1.  

````powershell
#Parse user object, create management name based on user attributes
$MSGUserResult = $MSGraphCall[1]
````

We will use hashtables to set the values for office, department, and city that will be used in the management name. 

````powershell
#Creat hashtables to assign short values based on Office, department, and city
$OfficeHash=@{"Dunder Mifflin" = "DMIF";"Pawnee Parks and Rec" = "PAWN";"Rosebud Motel" = "ROMO"}
$DepartmentHash=@{ "Payroll" = "CASH" ; "Overlords" = "BOSS" ; "Human Resources" = "HUMR"}
$LocationHash=@{"Houston" = "HOU" ; "Miami" = "MIA"; "Seattle" = "SEA"}

$User = $MSGUserResult.mailNickname
$Office = $officeHash.($MSGUserResult.officeLocation)
$Dept = $DepartmentHash.($MSGUserResult.department)
$Loc = $LocationHash.($MSGUserResult.city)
````

----

We have completed the steps required for the user object. We are ready to return the device object. This is very similar to returning the user object, but there are some additional quirks that make building the URI a bit more challenging. With the user object, we could return the specific user by UPN. To return a user’s devices, we need to return all managed devices and apply a filter. To add to the challenge, we need to account for special characters in the filter string. 
The full string for query parameters should be `?$filter=userPrincipalName eq '$userPrincipalName'&$select=id,deviceName,serialNumber,model'`. If we set `$DeviceFilter` to this value, it will try to interpret `$filter` and `$select` as variables. We can escape the `$` characters with a backtick character, and they will be interpreted as part of the string. When we make the call to Microsoft Graph, we will return the resulting value to `$Devices`.

````powershell
#Get all devices for user
#Build paramaters for returning user devices
$DeviceFilter = "?`$filter=userPrincipalName eq '$userPrincipalName'&`$select=id,deviceName,serialNumber,model"
$ManagedDevicesRoot = "https://graph.microsoft.com/v1.0/deviceManagement/managedDevices"
$URI = "$ManagedDevicesRoot$Devicefilter"
$Method = "GET"

#Call Invoke-MsGraphCall to return user's devices
$MSGraphCall = Invoke-MsGraphCall -AccessToken $AccessToken -URI $URI -Method $Method -Body $Body

#Create list of devices from Graph result
$Devices = $MSGraphCall.value
````

Running this portion of the script will return a list of the user’s devices. (I made an editorial decision to return the managedDeviceName instead of the id in the following screenshot for demo purposes).

![Users Devices Returned](https://managedblog.github.io/managed/assets/images/22.01.04/02.UsersDevicesReturned.png){: .align-center}
----

Finally, we can iterate through each device in the list. In our list of devices, we can see that most of the devices are virtual machines. VMs return a prohibitively long serial number, so it wouldn’t be practical to add this to the management name. One of the devices is not returning a serial number at all. I have added logic to my script to use the deviceName for VMs and devices without a serial number. We will add that value to the `$MgmtAttribute` variable. 

````powershell
ForEach($Device in $Devices){

    $serial = $Device.SerialNumber
    $model = $Device.model
    $DevName = $Device.deviceName
    $DevId = $Device.id

    #Check to see if device is virtual or has no serial number, if either are true, use device name in Management Name
    #If serial number exists and device is not virtual, use serial number in management name
    If(!$serial -or ($model -eq 'Virtual Machine')){

        $MgmtAttribute = $DevName

    } Else {

        $MgmtAttribute = $serial

    }
````

Now that we have all the elements of the management name, we can create a string and assign it to a variable.

````powershell
    #Create new management name
    $NewManagementName = "$User-$Office-$Dept-$Loc-$MgmtAttribute"
````

The parameters for our final API call are relatively straightforward. The method in this instance will be `PATCH`. The URI will be `https://graph.microsoft.com/beta/deviceManagement/managedDevices/$DevId`. In this case, we need to add a JSON payload. Create a hashtable using the format `@{ “key” = “value” }`. Convert it to JSON using the `ConvertTo-Json` cmdlet. 

````powershell

    #Update the management name of the device at the URI for each device ID
    $URI = "https://graph.microsoft.com/beta/deviceManagement/managedDevices/$DevId"
    $Body = @{ "managedDeviceName" = "$NewManagementName" } | ConvertTo-Json  
    $Method = "PATCH"

    #Call Invoke-MsGraphCall to update management name
    $MSGraphCall = Invoke-MsGraphCall -AccessToken $AccessToken -URI $URI -Method $Method -Body $Body

}
````
----

Our script is now complete. We can run it from a PowerShell console with the -UserPrincipalName switch and pass in the UPN of the user whose devices we would like to update.

![Run From Console](https://managedblog.github.io/managed/assets/images/22.01.04/03.RunFromConsole.png){: .align-center}

This script doesn’t return an output, but if we return our list of devices for this user with the managedDeviceName attribute selected, we can see that the devices were updated:

![Renamed Devices Returned](https://managedblog.github.io/managed/assets/images/22.01.04/04.RenamedDevicesReturned.png){: .align-center}

----

We have now successfully created a script to automate a simple device management task in Intune using Microsoft Graph. Once we defined the script, we were able to use tools like Graph Explorer and Developer Tools in Edge to identify the API calls we needed to make. We were able to streamline the results returned our various API calls using query parameters. Build a script was straightforward, especially once we created the Invoke-MsGraphCall function that was able to handle any API call, provided we passed in the proper parameters. 

The goal of this blog series is to provide the building blocks to work with various REST APIs to automate endpoint management tasks. I started with basic concepts and have progressed to building a PowerShell script that leverage Microsoft Graph. Future blog posts will discuss how to connect to the Configuration Manager AdminService, and how to leverage tools like Azure Automation and PowerAutomate to make your tools work even better for you.

______

Follow the full series below:

Follow the full series below:

1. [Everything I wanted to know about APIs but was afraid to ask](https://www.modernendpoint.com/managed/everything-i-wanted-to-know-about-apis-but-was-afraid-to-ask/)
2. [Connecting to Microsoft Graph with PowerShell](https://www.modernendpoint.com/managed/connecting-to-microsoft-graph-with-powershell/)
3. [Troubleshooting Microsoft Graph App Registration Errors](https://www.modernendpoint.com/managed/troubleshooting-microsoft-graph-app-registration-errors/)
4. [Defining a script and finding the Microsoft Graph Calls](https://www.modernendpoint.com/managed/Defining-a-script-and-finding-the-Microsoft-Graph-Queries/)
5. [Splatting with Invoke-RestMethod in PowerShell](https://www.modernendpoint.com/managed/PowerShell-tips-for-accessing-Microsoft-Graph-in-PowerShell/)
6. [Updating Device Management name in PowerShell with Microsoft Graph](https://www.modernendpoint.com/managed/Updating-device-management-name-in-PowerShell-with-Microsoft-Graph/)
7. [Surprise Post:Adding Filters to application assignments with PowerShell and Microsoft Graph](https://www.modernendpoint.com/managed/Adding-Filters-to-application-assignments-with-PowerShell-and-Microsoft-Graph/)
8. [Working with Azure Key Vault in PowerShell](https://www.modernendpoint.com/managed/Working-with-Azure-Key-Vault-in-PowerShell/)