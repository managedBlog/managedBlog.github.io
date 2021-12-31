---
title: "Splatting with Invoke-RestMethod in PowerShell"
excerpt: "Invoke-RestMethod is a much more flexible option. We can make any REST API call we want to, it is native to PowerShell, and it automatically converts JSON or XML responses to PowerShell objects."
tags:
  - Microsoft Endpoint Manager
  - Intune
  - Microsoft Graph API
  - AdminService
  - PowerShell
---


_<small>Hello! Welcome back to my series on automating endpoint management tasks with Microsoft Graph and the AdminService. [My last post]( https://www.modernendpoint.com/managed/Defining-a-script-and-finding-the-Microsoft-Graph-Queries/) covered how to find the information you need to build a working automation script. In today’s post, I am going to discuss the elements in PowerShell that we need to put that information to good use. In future posts I will continue to build on these concepts to provide a guide on building tools that leverage these APIs. [You can find the rest of the series here.](https://www.modernendpoint.com/tags/#microsoft-graph-api)</small>_


| ![header](https://managedblog.github.io/managed/assets/images/21.12.30/00.header.png){: .align-center} |


We already know that when we make a call to Microsoft Graph we need to be able to make an API call that includes the endpoint, method, and any information needed to complete our request. The next step is to take that information and be able to use it in a PowerShell script. This post will cover some important tips and tricks to get the most out of Microsoft Graph when working with PowerShell. There are a lot of different things I could cover, but as I have been working through this blog post, I wanted to break it down to three main areas: using the right version of PowerShell, splatting, and a primer understanding return codes.

### PowerShell Versions
____

Before I jump into the depths of today’s post, let’s start with what I’m not talking about here. There are a lot of different ways to accomplish any task. I made some editorial decisions not to cover certain topcs in today’s post for various reasons. Two that stand out to me are how to work with JSON and the Microsoft Graph SDK for PowerShell. They are both important topics, but I think they take away from the spirit of this post and blog series. The SDK is powerful but doesn’t support custom API calls. Invoke-RestMethod is a much more flexible option. We can make any REST API call we want to, it is native to PowerShell, and it automatically converts JSON or XML responses to PowerShell objects. I want to help you build custom tools that will work in any environment, and that makes Invoke-RestMethod the best option.

Invoke-RestMethod is part of the PowerShell.Utility module. If you’re following this series, you probably already imported the MSAL module, so we don’t need to import anything else to access Microsoft Graph. Invoke-RestMethod was updated in PowerShell version 7. The StatusCodeVariable parameter was added. This parameter allows us to easily pass the return code from our REST call back to our script. It’s not a requirement, but it will make troubleshooting and more robust scripts possible. If you aren’t already using PowerShell version 7, I highly recommend updating now.

[Invoke-RestMethod](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod?view=powershell-7.2) is used to send requests to Microsoft Graph  (or any other REST API). There are several different parameters available, but there are only a handful we will need to use regularly. I commonly use Method, Uri, Headers, Body, and StatusCodeVariable.

1.	Method – this is the verb that describes what we are trying to do with our API call.
2.	Uri – this is the endpoint (or URL) we are accessing
3.	Headers – the Header include information requiring in the request, including our authorization token and accepted response type
4.	Body – this is a JSON payload that is delivered as part of a PUT, PATCH, or POST call
5.	StatusCodeVariable – the status code variable tells PowerShell where to store the status code returned by the service

We can pass each one of these parameters to Invoke-RestMethod in a single line of code. That’s fine for calling Invoke-RestMethod from a terminal, but it’s going to be difficult to read in a script. It also means that we may have to make multiple API calls with explicitly defined parameters. This is cumbersome and doesn’t follow best practices for scripting. 

### Splatting
____

Splatting in PowerShell is used to pass parameters into a command using either a hashtable or an array. Arrays can be used to pass positional parameters, while hashtables can be used to pass named parameters. 

Splatting allows you to reuse code, which will help to keep your scripts cleaner and easier to read. If you reuse splats with common parameters, you will be able to consistently pass common parameters and minimize errors. Parameters can be updated or added to a splat at any time prior to making an API call. 

A simple splat for a GET request could look like this:

````powershell
$graphGetParams = @{

    Headers     = @{
        "Content-Type"  = "application/json"
        "Authorization" = "Bearer $($AccessToken)"
    }
    Method      = "GET"
    URI = "https://graph.microsoft.com/v1.0/users"
    StatusCodeVariable = "SCV"
    ErrorAction = "SilentlyContinue"

}
````

When we create a splat, we are creating a hashtable using the `$variable = @{ key = “value” }` format. Each parameter we want to add to the splat is used as a key in our hashtable, and the values are set accordingly. We can call Microsoft Graph using the following format:

````powershell
$Result = Invoke-RestMethod @graphGetParams 
````

The result is returned as a PSCustomObject.

![Result Returned As Object](https://managedblog.github.io/managed/assets/images/21.12.30/01.ResultReturnedAsObject.png){: .align-center}

We can view the results returned from the API call by calling the `$_.value` property of the object. Since it is a PowerShell object, we can even pipe the value to another command. In this case, I wanted to view all the users returned that had a DisplayName that starts with “lab.”

````powershell
$result.value | Where-Object DisplayName -like 'lab*'
````

![result Piped To Where Object](https://managedblog.github.io/managed/assets/images/21.12.30/02.resultPipedToWhereObject.png){: .align-center}

We also added the `StatusCodeVariable` parameter to our splat. By calling the variable we set, in this case `$SCV`, we can view the status code that was returned.

![Status Code200](https://managedblog.github.io/managed/assets/images/21.12.30/03.StatusCode200.png){: .align-center}

We can add additional parameters or update existing parameters in a splat by using the format `$SplatVariable[“Key”] = “New Value”`. We can see that in the following example:

We will start by creating a hashtable without the URI key.

````powershell
$graphGetParams = @{

    Headers     = @{
        "Content-Type"  = "application/json"
        "Authorization" = "Bearer $($AccessToken)"
    }
    Method      = "GET"
    StatusCodeVariable = "SCV"
    ErrorAction = "SilentlyContinue"

}
````

The hashtable was created without the key:

![hashtable With No Key](https://managedblog.github.io/managed/assets/images/21.12.30/04.hashtableWithNoKey.png){: .align-center}

We can add a URI to the hashtable with the following command:

` $graphGetParams["URI"]= “https://graph.microsoft.com/beta/users”`

When we return the variable, we can see the URI has been added to our hashtable.

![Uri Added To Hashtable](https://managedblog.github.io/managed/assets/images/21.12.30/05.UriAddedToHashtable.png){: .align-center}

We can update a value using the exact same command:

` $graphGetParams["URI"]= “https://graph.microsoft.com/v1.0/devices”`

Here we can see that the value was updated:

![Uri Updated](https://managedblog.github.io/managed/assets/images/21.12.30/06.UriUpdated.png){: .align-center}

There are different ways we can approach using splatting in PowerShell scripts. One approach is to create a splat for each type of API call we will make at the start of our script, and then call the specific splat when we need to call Invoke-RestMethod. If we take this approach, we will need to have multiple splats declared early in our script. We will also need to add or update the URI and Body each time we use a splat. For example, if we have a script that will be making GET, POST, and PATCH calls, we could include the following section in our script:

````powershell
#Create MS Graph Splats
#Create GET splat
$graphGetParams = @{
    Headers     = @{
        "Content-Type"  = "application/json"
        "Authorization" = "Bearer $($AccessToken)"
    }
    Method      = "GET"
    ErrorAction = "SilentlyContinue"
    StatusCodeValue = "SCV"
}

#Create POST splat
$graphPostParams = @{
    Headers     = @{
        "Authorization" = "Bearer $($AccessToken)"
        "Accept"        = "application/json"
        "Content-Type"  = "application/json"
    }
    Method      = "POST"
    ErrorAction = "SilentlyContinue"
    StatusCodeValue = "SCV"
}

#Create PATCH splat
$graphPatchParams = @{
    Headers     = @{
        "Authorization" = "Bearer $($AccessToken)"
        "Content-Type"  = "application/json"
    }
    Method      = "PATCH"
    ErrorAction = "SilentlyContinue"
    StatusCodeValue = "SCV"
}
````

If we take this approach, we can add the additional parameters to the splat immediately before running Invoke-RestMethod. For example, if we want to return all Azure AD Users, we can use the following script to connect to Microsoft Graph, create a splat hashtable, and finally call Invoke-RestMethod:

````powershell
#Use a client secret to authenticate to Microsoft Graph
$authparams = @{
    ClientId    = '1234xxxx-12xx-34xx-9876-12345xxxxx'
    TenantId    = 'modernendpoint.xyz'
    ClientSecret = ('No-CLIENT-SECRET-FOR-YOU' | ConvertTo-SecureString -AsPlainText -Force )
}

$auth = Get-MsalToken @authParams

$AccessToken = $Auth.AccessToken

#Create MS Graph Splats
#Create GET splat
$graphGetParams = @{
    Headers     = @{
        "Content-Type"  = "application/json"
        "Authorization" = "Bearer $($AccessToken)"
    }
    Method      = "GET"
    ErrorAction = "SilentlyContinue"
    StatusCodeValue = "SCV"
}

$graphGetParams["URI"]= “https://graph.microsoft.com/beta/users”

$Result = Invoke-RestMethod @graphGetParams

$Result
````

This will return our users to a `PSCustomObject` as we demonstrated in the first example.

![Users Returned From Splat](https://managedblog.github.io/managed/assets/images/21.12.30/07.UsersReturnedFromSplat.png){: .align-center}

Perhaps the better option would be to include a splat as part of a function. PowerShell functions allow us to reuse code, so we could simply create a different function for each method, but we can take it a step further. By passing in parameters for the URI, method, and body, we can use a single function to handle every method available to us in Microsoft Graph. 

````powershell
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
````

We can adjust the Splat in the function to account for different methods. In this case, both the method and URI values are variables. Their values are being passed in from the function’s parameters.

````powershell
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
````

Some methods require a body, but not all of them do. We want to be able to add the body to our splat, but only if it is needed. We can accomplish that with a conditional statement that looks for methods that require a body.

````powershell
    #If method requires body, add body to splat
    If($Method -in ('PUT','PATCH','POST')){

        $graphSplatParams["Body"] = $Body

    }
````

Finally, we will make our API call using Invoke-RestMethod. Any value returned will be returned to the $MSGraphResult variable. That value and our status code variable will be returned to the main script.

````powershell
    #Return API call result to script
    $MSGraphResult = Invoke-RestMethod @graphSplatParams

    #Return status code variable to script
    Return $SCV, $MSGraphResult

````

Now that we have created our function (the full function is included in the script below and [available in my GitHub repository](https://github.com/managedBlog/Managed_Blog/blob/main/Microsoft%20Graph/Splatting/Invoke-MsGraphCall.ps1)), let’s test it out. In my last post, we identified that we want to be able to update a device’s management name as part of our first tool. We know that we can update a managed device name with a  `PATCH` request to `https://graph.microsoft.com/beta/deviceManagement/managedDevices([DeviceID]')`.

We need to include the new name in the body of the request, so we will have to pass the body to the function through a parameter.

We can see that the device we worked with in the last post still has the Management name “Renamed_Device_Final.” 

![Original Device Name](https://managedblog.github.io/managed/assets/images/21.12.30/08.OriginalDeviceName.png){: .align-center}

I have created a script that includes the function outlined above. This script authenticates to MS Graph using an app registration and a client secret. Each value we need to pass into our function is saved to a variable, and each variable is being passed to the function.

The `BODY` parameter needs to be passed to Invoke-RestMethod as JSON. When we create the `$Body` variable, we create a hashtable and pipe that to ConvertTo-JSON as shown in the script below.

````powershell
#Use a client secret to authenticate to Microsoft Graph using MSAL
$authparams = @{
    ClientId    = '1234xxxx-5678-90xx-1234-12345xxxxx'
    TenantId    = 'YourTenantId.xyz'
    ClientSecret = ('NoClientSecretForYou!' | ConvertTo-SecureString -AsPlainText -Force )
}

$auth = Get-MsalToken @authParams

#Set Access token variable for use when making API calls
$AccessToken = $Auth.AccessToken

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

#Create required variables
$URI = "https://graph.microsoft.com/beta/deviceManagement/managedDevices('1234xxxx-5678-90xx-1234-12345xxxxx ')"
$Body = @{ "managedDeviceName" = "Renamed_Via_FunctionSplat" } | ConvertTo-Json
$Method = "PATCH"

#Call Invoke-MsGraphCall
$MSGraphCall = Invoke-MsGraphCall -AccessToken $AccessToken -URI $URI -Method $Method -Body $Body

$LastStatusCode = $MSGraphCall[0]
$ReturnedValue = $MSGraphCall[1].value

$ReturnedValue
Write-Host "SCV is $LastStatusCode"
````

The `PATCH` method doesn't return a value, so there is no value for `$ReturnedValue`, but a status code value was returned. We can see that in our terminal after running the script. `204` is the exit code that we would expect for a successful `PATCH` request.

![SCV is 204](https://managedblog.github.io/managed/assets/images/21.12.30/09.SCVis204.png){: .align-center}

We can also see that our device was renamed as we expected:

![Device Renamed In Portal](https://managedblog.github.io/managed/assets/images/21.12.30/10.DeviceRenamedInPortal.png){: .align-center}


### Microsoft Graph Response Codes
____

When we make calls to Microsoft Graph, we always expect a response code, whether the API call was successful or not. These response codes are essential for understanding whether our call was accepted and processed successfully. If the API call failed, the response code can be essential to understand what went wrong and how to correct it. The following table lists some of the most common error codes you may encounter:

Status Code | Summary | Notes
:----:|:-----|:-----
200 OK | Request Accepted | The request was accepted and the response contains a result. This is the expected response for GET requests, but can be returned by other requests.
204 NO CONTENT |Accepted, no content returned | The request was accepted, but the service didn’t return any response content. This is an expected response for calls other than GET.
401 UNAUTHORIZED | The authorization headers are missing | This error code generally means that the authorization headers were missing or incomplete.  It could also indicate that authentication failed.
403 FORBIDDEN | The application attempted to access resources it didn’t have access to. | Ensure the user or app registration has the appropriate permissions assigned.
404 NOT FOUND | The resource does not exist | Check the URI and confirm the endpoint exists
500 INTERNAL SERVER ERROR | A server-side error occurred | There may be an issue with the request, or there is a server side error. The response body may contain more information. 
503 SERVICE UNAVAILABE | The server did not respond | The server didn’t respond to the incoming request. 


[You can find out more information about status codes here.](https://docs.microsoft.com/en-us/graph/errors) As we work through this series, I will build error handling into my scripts. Since we are creating tools for automation, a lot of the errors will simply be written to a log file, but when appropriate we may take different actions based on the status code returned. Even if you’re not building in error handling, it’s important to understand the return codes to help identify areas where your API calls can be improved.

This blog post includes tips and tricks for making API calls with Invoke-RestMethod in PowerShell. We discussed splatting and created a function that can be used for calling Microsoft Graph. In my next post, I will combine the topics from both this post and my previous post to create a script that will automatically update a device’s management name based on the primary user. You will see how we can use PowerShell to parse the results of a GET request, dynamically create a name string, and then make a PATCH request to set the name. It will demonstrate the power of the function we created today and demonstrate how Microsoft Graph can be used to automate tasks across Azure services.


______

Follow the full series below:

1. [Everything I wanted to know about APIs but was afraid to ask](https://www.modernendpoint.com/managed/everything-i-wanted-to-know-about-apis-but-was-afraid-to-ask/)
2. [Connecting to Microsoft Graph with PowerShell](https://www.modernendpoint.com/managed/connecting-to-microsoft-graph-with-powershell/)
3. [Troubleshooting Microsoft Graph App Registration Errors](https://www.modernendpoint.com/managed/troubleshooting-microsoft-graph-app-registration-errors/)
4. [Defining a script and finding the Microsoft Graph Calls](https://www.modernendpoint.com/managed/Defining-a-script-and-finding-the-Microsoft-Graph-Queries/)