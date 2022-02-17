---
title: "Defining a Script and Finding the Microsoft Graph Queries"
excerpt: "Over the course of the next few posts, I am going to create a script that will update a device’s management name based on the primary user. "
tags:
  - Microsoft Endpoint Manager
  - Intune
  - Microsoft Graph API
  - AdminService
  - PowerShell
---

_<small>Welcome back! This post is the fourth post in my series on Automating Endpoint Manager tasks using Graph API and the AdminService. This post will be the first one that will walk through creating a PowerShell script to automate a task with Microsoft Graph. In this post I will discuss how to find the information needed to create your script. [You can find the rest of the series here.](https://www.modernendpoint.com/tags/#microsoft-graph-api)</small>_


Defining a Script and Finding the Microsoft Graph Queries
-------

Hello!

This series is intended to help provide the building blocks to automate device management workloads in your environment using various APIs. In previous posts, I discussed the basics of REST APIs, how to authenticate to Microsoft Graph, and how to troubleshoot authentication to Microsoft Graph. The next step is to define the objective for a script and identify the API calls we need to make to perform those tasks.

One of the most common questions I get from customers is how to apply a device naming template to an Intune device with Autopilot. The built-in naming templates are much more limited than most organizations are used to. We can specify a device name using the Autopilot devices list, but that process is manual and only works for cloud-only devices. Our options are even further limited for hybrid joined devices.

We can look for alternatives to change a device’s name, but as we shift to a modern device management mindset it may be better to look for alternatives. One option would be to change a device’s management name. The default management name isn’t the most useful name in most organizations. It has the username, device type, and a timestamp. However, if we change the management name, we can tailor it to fit our specific needs. It is searchable in the Intune portal, so we can shift our naming conventions to the management name with little to no impact on our daily operations.

Over the course of the next few posts, I am going to create a script that will update a device’s management name based on the primary user. The name will be changed based on the user name, department, and location. To do this, I will need to complete the following tasks:

1.	GET the device object from Azure Active Directory
2.	GET the primary user’s Azure AD object
3.	Create a payload from the response
4.	PATCH the device object to change the management name


We need to find the URI to do each one of these items. There are several different options, but I want to call out three different options: [Microsoft Graph REST API documentation](https://docs.microsoft.com/en-us/graph/api/overview?view=graph-rest-1.0), Graph Explorer (aka.ms/ge), and using F12 Developer Tools to identify the API being called by the UI.


### Microsoft Graph REST API Documentation 
____

Microsoft Graph is extremely well documented. Microsoft's documentation explores all the available endpoints in depth. It is arguably the most important tool for understanding how to work with Microsoft Graph. The single greatest weak spot of the Microsoft Graph documentation is the search tool. You can search from the search box in the upper right corner and filter the results, but it's a bit clunky. The best way I have found to search the documentation is by using an external search engine. In this case, I am going to search for "Microsoft graph management name." The second result is from Microsoft Docs, and is titled "List managedDevices - Microsoft Graph v1.0."

![Bing Search](https://managedblog.github.io/managed/assets/images/21.12.19/01.BingSearch.png){: .align-center}

Clicking on this link takes us to the Microsoft Docs page for the API call to `GET managedDevices`. Microsoft Docs covers each API call in depth. This page covers the required API permissions that are needed to make the API call, the syntax for the request, required body and header information, and finally it covers the expected response. We can explore different API calls by using the table of contents on the left side of the page to navigate the documentation.

![Docs List Managed Devices](https://managedblog.github.io/managed/assets/images/21.12.19/02.DocsListManagedDevices.png){: .align-center}

`GET managedDevices` will return a list of all managed devices. At the bottom of the page we see an example response. The response starts with the query results, in this case it was `200 OK.` This means the query was valid and a result was returned. The `Content-Type` is `application/json`, which tells us the data being returned is in json format. Json is a standard file format that stores data in a structure of attribute-value pairs. The data can be complex, including both objects and arrays of objects. 

If we look at the example, we can see brackets being used to define the data. Data stored in curly brackets `{}` constitutes an array, while data stored in square brackets `[]` is a single object within that array. In this example the first character returned is a curly bracket, which tells us that the response contains multiple objects. The first object in the array of objects (value), is defined as a single object with the square brackets. 


![Docs Example1](https://managedblog.github.io/managed/assets/images/21.12.19/03.DocsExample1.png){: .align-center}

Further down the response, we see that we also have a value for `managedDeviceName`. Based off our requirements, we know that this is the value that we will want to update in our script.

![Docs Example Managed Device Name](https://managedblog.github.io/managed/assets/images/21.12.19/04.DocsExampleManagedDeviceName.png){: .align-center}

Ultimately, this API call returned a list of objects. When we build our script later, we will need to update a single object. We can identify a single object by its id, which we can see in the response here. 

![Docs Example Id](https://managedblog.github.io/managed/assets/images/21.12.19/05.DocsExampleId.png){: .align-center}


### Graph Explorer 
____

Microsoft Graph documentation includes a lot of great information, though it can be difficult to find exactly what you’re looking for. In the first post in this series, I talked about Graph Explorer at a high level. It is an extremely valuable tool for understanding how to make calls to Microsoft Graph. 

After accessing Graph Explorer and connecting to the service we can make API calls against our tenant. We can use the address bar to explore Microsoft Graph. REST APIs have navigable properties, and Graph Explorer uses autocomplete to make discovering those navigable properties easier. In this case, we removed the /me from the address bar. The address bar will give us an autocomplete list that includes all the different endpoints that are available. Intune managed devices are in deviceManagement in Microsoft Graph. 

![Autocomplete In Graph Explorer](https://managedblog.github.io/managed/assets/images/21.12.19/06.AutocompleteInGraphExplorer.png){: .align-center} 

When we select deviceManagement, we can explore deeper. We see managedDevices, the same endpoint we discussed above, is available. 

![Autocomplete Managed Devices](https://managedblog.github.io/managed/assets/images/21.12.19/07.AutocompleteManagedDevices.png){: .align-center}  

The examples that I used for Microsoft Docs are from the results listed in the example documentation. When we return information using the Graph Explorer, we are returning live data from my lab environment. In this case we see similar results get returned. Note that in this case, the `@odata.count` value is `8`. This is the number of items returned in the results.

![Managed Device Results](https://managedblog.github.io/managed/assets/images/21.12.19/08.ManagedDeviceResults.png){: .align-center}

When making calls to Microsoft Graph we can use query parameters to tailor our response to meet our specific needs. For example, we can use a select query to return specific attributes of an object. We can create a select query by appending our API call with `?$select=` and then entering the attribute names we would like to return. Autocomplete will also show us the attributes that we can return from the endpoint. In this case our full API call is `https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$select=id,deviceName,managedDeviceName`

![Managed Device Select Query](https://managedblog.github.io/managed/assets/images/21.12.19/09.ManagedDeviceSelectQuery.png){: .align-center}  

Here we can see that our results include only the attributes we added to our select query. By selecting only specific attributes we can remove the noise created by returning additional attributes and control the amount of data being returned in the response. Each managed device has over 50 attributes. If we were to run this same query with no query parameters the amount of data would be huge, and that can have an impact on how efficiently automation workloads can run at scale.

![Results With Select Query](https://managedblog.github.io/managed/assets/images/21.12.19/10.ResultsWithSelectQuery.png){: .align-center}

There are several different query parameters that can be used when making API calls to Microsoft Graph. [You can read about all of them here.](https://docs.microsoft.com/en-us/graph/query-parameters)

This post won’t explore all of the possible query parameters, but one other query I regularly use is the `filter` parameter. To use filter, append your API call with `$filter=` and then build a filter using the correct filter syntax, [which is discussed in depth here.](https://docs.microsoft.com/en-us/graph/query-parameters). You can also use multiple query parameters. In this example, I am filtering devices by `userPrincipalName` and selecting only the attributes I want to return. 

`https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$filter=userprincipalname eq 'sean.bulger@modernendpoint.xyz'&$select=id,deviceName,managedDeviceName,userPrincipalName`

![Filtered And Selected Results](https://managedblog.github.io/managed/assets/images/21.12.19/11.FilteredAndSelectedResults.png){: .align-center}   

### F12 Developer Tools
____

We can find all the information we need to write our script using Microsoft Docs and Graph Explorer, but there is one more tool that can prove to be invaluable when working with Microsoft Graph. Most (if not all) modern browsers have a developer tools feature built in. Developer tools can inspect web pages and traffic to provide information on a site and the resources being accessed. In Microsoft Edge and internet Explorer developer tools are accessed by pressing F12. 

When you are working with Microsoft Graph and want to understand how a page gets the data it is displaying, or how a specific field is being updated you can load developer tools to see the API calls being made.

In the following example I am going to use developer tools to find the API call that is being made to update the management name on a device. This can be done on a device’s properties page in Microsoft Endpoint Manager Admin Center. 


![Viewing Device In Intune](https://managedblog.github.io/managed/assets/images/21.12.19/12.ViewingDeviceInIntune.png){: .align-center}

We can search for how to update a device’s management name through Microsoft Docs. We can probably even use Graph Explorer to piece together the information needed to make the API call, but what if there were a way to see the exact API call being made by this page? It could potentially save a lot of time and help us to understand how to use specific calls. 

This is where developer tools come in. In Microsoft Edge you can load developer tools by pressing F12. Developer tools will open to a welcome screen in a separate pane.

![F12 Developer Tools](https://managedblog.github.io/managed/assets/images/21.12.19/13.F12DeveloperTools.png){: .align-center}

Some options may not be visible based on where DevTools opens in your browser. In Edge it’s a flyout on the right side by default, but you can move the dock location by clicking the ellipses and selecting a different location. If you keep the tools docked on the right side, click `>>` to view other menu options. Click on Network to open the network tools.

![Select Network](https://managedblog.github.io/managed/assets/images/21.12.19/14.SelectNetwork.png){: .align-center}

On the network tab we can see all the various network traffic from the website. We want to view the specific API call being made to Microsoft Graph, so we are going to enter graph.microsoft.com in the filter box. 

![Filter By Graph](https://managedblog.github.io/managed/assets/images/21.12.19/15.FilterByGraph.png){: .align-center}

On the device properties page, enter a new management name and click save.

![Rename In U I](https://managedblog.github.io/managed/assets/images/21.12.19/16.RenameInUI.png){: .align-center}

After clicking save, we should see traffic appear in the network tab of developer tools. In this instance, we see one call that was made to graph with the name “managedDevices.”

![A P I Call In Console](https://managedblog.github.io/managed/assets/images/21.12.19/17.APICallInConsole.png){: .align-center}

Clicking on managedDevices will show us the information included in this request. We can see information like the API call, the method, response, and request and response headers.

![Patch Request In Dev Tools](https://managedblog.github.io/managed/assets/images/21.12.19/18.PatchRequestInDevTools.png){: .align-center}

In this case, we can see that the API call made to Microsoft Graph was a PATCH request to the URL `https://graph.microsoft.com/beta/deviceManagement/managedDevices('9xxxxxxx-xxxx-xxxx-x-xxxxxxxxxx44')`.

If we click on the Payload tab, we can see the actual request body that was passed to the service.

![Payload In Dev Tools](https://managedblog.github.io/managed/assets/images/21.12.19/19.PayloadInDevTools.png){: .align-center}

We can combine these details to make an API call in PowerShell. In fact, developer tools make it even easier to take an API call and run it in PowerShell. We can copy the PowerShell cmdlet directly from the browser and run in PowerShell. Right click on the API call in the browser, select copy, and click on “Copy as PowerShell.”

![Copy As Powershell](https://managedblog.github.io/managed/assets/images/21.12.19/20.CopyAsPowershell.png){: .align-center}

Paste the code into your favorite code editor. Here we can see that the request includes our URI, method, and body. We can also see the headers, which include the authorization token. The token from the browser will remain usable until it is no longer valid, so we can run the query directly in PowerShell without requesting a new token.

![Power Shell In Console](https://managedblog.github.io/managed/assets/images/21.12.19/21.PowerShellInConsole.png){: .align-center}

Because this was a patch request, we are going to update the name in the body of the request. 

![Changed Name In Powershell](https://managedblog.github.io/managed/assets/images/21.12.19/22.ChangedNameInPowershell.png){: .align-center}

Running the request returns a response of 204.

![Response Restored](https://managedblog.github.io/managed/assets/images/21.12.19/23.ResponseRestored.png){: .align-center}

If we look in Microsoft Endpoint Manager, the management name was successfully updated.

![Device Name Changed In Portal](https://managedblog.github.io/managed/assets/images/21.12.19/24.DeviceNameChangedInPortal.png){: .align-center}    

### Pulling it back together
____

We defined that our goal was to update a device’s management name based on the primary user. That means we need to be able to complete the following steps:
1. Return a device object
2. Get the user object
3. Create a payload based on the user object
4. Patch the device object

Depending on what we want to accomplish with our script there are different ways to approach how we return a device object. We established above that we can return a single device by making a GET request to `https://graph.microsoft.com/v1.0/deviceManagement/managedDevices/[MANAGED-DEVICE-ID]`. If we know a specific device we want to update, this is a great option. We could also find a specific device by using the `$filter` query parameter. For example, we could return a device by deviceName using `https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$filter=deviceName eq ‘[DeviceName]’`. If we wanted to update all the devices for a specific user, we could filter by the UserPrincipalName.

In the interest of keeping things simple, we will update a single device. In this case, we want to update the device “DESKTOP-6A51HPA.” We will make a GET request to `https://graph.microsoft.com/v1.0/deviceManagement/managedDevices?$filter=deviceName eq 'DESKTOP-6A51HPA’&$select=deviceName,id,userPrincipalName`. This will return our device and the primary user, which we will use in the next step of our script.

First, we need to connect to Microsoft Graph. We will connect using the Microsoft Authentication Library. We will use a client secret so we can connect programmatically. You can read more about this in my post on [connecting to Microsoft Graph with Powershell](https://www.modernendpoint.com/managed/connecting-to-microsoft-graph-with-powershell/).

```powershell
#Use a client secret to authenticate to Microsoft Graph
$authparams = @{
    ClientId    = '3d81dbdd-7482-4a1e-8086-520ddf4e4d12'
    TenantId    = 'modernendpoint.xyz'
    ClientSecret = ('[Client Secret]' | ConvertTo-SecureString -AsPlainText -Force )
}
$auth = Get-MsalToken @authParams
```

After connecting to Graph, we can run the following script to return our device. 

```powershell
#Get access token from authentication results
$AccessToken = $Auth.AccessToken

#Create Splat for Microsoft Graph GET request
$graphGetParams = @{
    Headers     = @{
        "Content-Type"  = "application/json"
        "Authorization" = "Bearer $($AccessToken)"
    }
    Method      = "GET"
    ErrorAction = "SilentlyContinue"
    URI = 'https://graph.microsoft.com/beta/deviceManagement/managedDevices?$filter=deviceName eq ''DESKTOP-6A51HPA''&$select=deviceName,id,userPrincipalName'
}

#Make API call to GET Azure device
$Result = Invoke-RestMethod @graphGetParams 
$Result.value
```

This returns the following result:

![PS Returned User Object](https://managedblog.github.io/managed/assets/images/21.12.19/24b.PSReturnedUserObject.png){: .align-center}

Once we have the device, we can return the user object. Using Graph Explorer, I have found that I can return an Azure AD user object using `https://graph.microsoft.com/v1.0/users/[userPrincipalName]`. If we return a user using this API call, only some attributes are returned.

![Returning User In Graph Explorer](https://managedblog.github.io/managed/assets/images/21.12.19/25.ReturningUserInGraphExplorer.png){: .align-center}

We can select additional attributes using a select query parameter. Using Graph Explorer, we can see the list of selectable attributes is quite large.

![Select Attributes Autocomplete](https://managedblog.github.io/managed/assets/images/21.12.19/26.SelectAttributesAutocomplete.png){: .align-center}

When I set the management name on my devices, I want them to be specific to my user based on their city, office, department, and office location. To gather this information, I am going to use the following URL in my GET request: 
`https://graph.microsoft.com/v1.0/users/sean.bulger@modernendpoint.xyz?$select=displayName,officeLocation,city,usageLocation,department`

![User With Selected Attributes](https://managedblog.github.io/managed/assets/images/21.12.19/27.UserWithSelectedAttributes.png){: .align-center}   

We can run this directly in PowerShell with the following script:

```powershell
#Get access token from authentication results
$AccessToken = $Auth.AccessToken

#Create Splat for Microsoft Graph GET request
$graphGetParams = @{
    Headers     = @{
        "Content-Type"  = "application/json"
        "Authorization" = "Bearer $($AccessToken)"
    }
    Method      = "GET"
    ErrorAction = "SilentlyContinue"
    URI = 'https://graph.microsoft.com/v1.0/users/sean.bulger@modernendpoint.xyz?$select=displayName,officeLocation,city,usageLocation,department'
}

#Make API call to GET Azure user object
$Result = Invoke-RestMethod @graphGetParams 
$Result.value
```

This will return the following result:

![User Object Returnedfrom P S](https://managedblog.github.io/managed/assets/images/21.12.19/28.UserObjectReturnedfromPS.png){: .align-center}    

This post is focused on gathering the API calls that we will need to build our script. In my next post I will cover how to work with the results. The information we need to build our management name is included in these results. We will parse the JSON, build the name string, and create the payload that we will use to change the device’s management name. 

We already discovered the format of our payload up above. In this case, we know that we can use a PATCH request to update the management name on a device. The last API call we will need to make in our script will be:

```powershell
#Get access token from authentication results
$AccessToken = $Auth.AccessToken

#Create Splat for Microsoft Graph PATCH request
$graphPatchParams = @{
    Headers     = @{
        "Authorization" = "Bearer $($AccessToken)"
        "Content-Type"  = "application/json"
    }
    Method      = "PATCH"
    ErrorAction = "SilentlyContinue"
    URI         = "https://graph.microsoft.com/beta/deviceManagement/managedDevices('91092110-82e1-4028-ac71-91590e2d1944')"
    Body        = @{ "managedDeviceName" = "Renamed_Device_Final" } | ConvertTo-Json
}

#Make PATCH Request
Invoke-RestMethod @graphPatchParams 
```

When we run this request, we can see the result in the Intune Portal:

![Device Renamed In Portal Final](https://managedblog.github.io/managed/assets/images/21.12.19/29.DeviceRenamedInPortalFinal.png){: .align-center} 

This post covers various ways to discover the API calls you need to build a script to build automation workloads with Microsoft Graph. There are different ways to find information. Each has its own benefits. In my next post, I will take the information that we found to build a script that will update a device’s management name. We will discuss how to work with the JSON results of an API call, form the body of a request, and how to format different API calls in PowerShell. 

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