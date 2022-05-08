---
title: "Comparing Invoke-RestMethod to the PowerShell SDK"
excerpt: "For those of you who have worked with Microsoft Graph in PowerShell, you may have noticed one glaring omission in all those blog posts."
tags:
  - Microsoft Endpoint Manager
  - Intune
  - Microsoft Graph API
  - PowerShell
---
_<small>Welcome back! A lot has happened since my last post. This post is a companion to one of my sessions at MMSMOA. It is also the latest post in my series on automating endpoint management tasks with Microsoft Graph and the MEMCM AdminService. [You can find the rest of the series here.](https://www.modernendpoint.com/tags/#microsoft-graph-api)</small>_
----
![Header](https://managedblog.github.io/managed/assets/images/22.03.09/IMG_2978.png){: .align-center}

----

Hello and welcome back!

It feels like a lifetime since I last shared a blog post. I knew that there would be a large delay while I worked on MMS session prep, I just didn’t think it would be this long. A lot of things have happened since my last blog post. MMS was amazing. I learned so much and had the opportunity to engage with the entire community on a huge scale. The closest I have come to that scale was at MMS Miami Beach, and while that was an amazing conference, there is no comparison between the full MOA event on the smaller events. My week at MMS started with finding out that I had been awarded my first Microsoft MVP award. I am just starting to unpack what that means for me after a busy week. I will share more about both MMS and earning my MVP in future blog posts.

This post is a continuation of my series on automation with Microsoft Graph and the Configuration Manager AdminService. It is also meant as a companion to one of my MMS Sessions, “Microsoft Graph and PowerShell: A Deep Dive into Connecting to and working with Microsoft Graph.” Greg Ramsey was gracious enough to present with me. Working with Greg was amazing. He is an excellent mentor and a treasure of the MEMCM community. 

If you've been following [my current blog series](https://www.modernendpoint.com/tags/#microsoft-graph-api), you know that I've covered Microsoft Graph extensively over the last several months. For those of you who have worked with Microsoft Graph in PowerShell, you may have noticed one glaring omission in all those blog posts. My blog focuses on using `Invoke-RestMethod` to make direct API calls to Microsoft Graph. While that is _my_ preferred way to make calls to Graph, it is not the only way. The [Microsoft Graph PowerShell SDK](https://docs.microsoft.com/en-us/powershell/microsoftgraph/installation) is a collection of modules designed for interacting with Graph directly through PowerShell.

In the interest of full disclosure, when I started writing this post before MMS, I had every intention of making the case that anyone who is serious about working with Microsoft Graph in PowerShell should take the time to learn how to use `Invoke-RestMethod` to make direct API calls. While I still believe that is the better option (more on that below), I was pleasantly surprised with the SDK as I spent more time working with it. It has some limitations, but if all you’re working with is Microsoft Graph in a PowerShell script or console, it is an excellent option.

All of the scripts that are referenced in this post can also be found in [my GitHub](https://github.com/managedBlog/Managed_Blog/tree/main/MMSMOA%202022%20Demos/Access%20MS%20Graph%20through%20PowerShell).

### The SDK: What is it, and why should I use it?
-----

The Microsoft Graph PowerShell SDK consists of a main module and 38 sub modules.  The SDK is essentially a wrapper that allows you to use PowerShell cmdlets to make calls to Graph. It is targeted as Systems Administrators who are familiar with PowerShell and may not have as much experience working with various APIs. It is a great option. It is fairly well documented, even though it’s not always easy to find the cmdlets you need in the documentation. It does have built in tools to allow you to discover the cmdlets you need in your scripts, and that feature makes up for a lot of the shortcomings in the documentation.

Microsoft has made a lot of entry points available to discover PowerShell cmdlets. For example, if we look at the [Microsoft Graph docs page for returning a user](https://docs.microsoft.com/en-us/graph/api/user-get?msclkid=8008e797ceff11ecb07ef6ea15cee974&view=graph-rest-1.0&tabs=powershell) from Microsoft Graph, there is an option to view the PowerShell cmdlets. It even shows which SDK module we need to import in the sample.


![Microsoft Docs Get User](https://managedblog.github.io/managed/assets/images/22.05.08/01.MicrosoftDocsGetUser.png){: .align-center}
 
 
[Microsoft Graph Explorer](https://aka.ms/ge) also includes PowerShell on the code snippets tab in the response. Both options make finding the required PowerShell cmdlets and finding examples extremely easy.


![Graph Explorer Code Snippets](https://managedblog.github.io/managed/assets/images/22.05.08/02.GraphExplorerCodeSnippets.png){: .align-center}
 

Finally, [the SDK documentation](https://docs.microsoft.com/en-us/powershell/microsoftgraph/overview?view=graph-powershell-beta) has a wealth of knowledge on the individual cmdlets, but because the cmdlet names can be overwhelming it can be difficult to navigate.

If you are new to Microsoft Graph and REST APIs the SDK is a great jumping off point. It is well documented and has a very familiar look and feel. There's a low barrier to entry and it doesn't require learning a new skill set. You can build automation workflows or perform one-off tasks with relative ease. 


### Using the SDK in PowerShell
----

Now that we have covered why we would use the SDK, let's talk a little bit about how we use it. Like any module, you will need to install the SDK. The SDK requires PowerShell 5.1 or later. As previously mentioned, there are 38 sub-modules in the SDK. You don't have to install all of them. You can choose to only install the ones that are required for your tasks. One of the most important features of Microsoft Graph is the ability to work across a wide range of Microsoft services, so I prefer to install the entire module.
 
Install the module by opening PowerShell as administrator and running `Install-Module Microsoft.Graph`.
 
After installing the module, we will need to sign in to use Graph. You can switch between the graph endpoint versions by using the cmdlet `Select-MgProfile` and calling the `-Name` parameter with either `v1.0`, or `beta`.  The Microsoft Graph SDK supports either interactive or certificate-based authentication. Authentication with a client secret is not supported. When connecting to the SDK interactively you also need to pass in the required permissions. When you first connect using `Connect-MgGraph` it will create a service principal in your environment. Whenever you connect with additional permissions, those permissions will also be provisioned for the service principal. It's important to note that this could lead to a service principal that amasses many permissions over time.
 
If we want to perform management tasks on users and their devices in Intune, we can connect with the following command:


````powershell
 
#Connect to graph interactively with requested scopes.
Connect-MgGraph -Scopes "User.ReadWrite.All","Directory.ReadWrite.All","DeviceManagementManagedDevices.ReadWrite.All"
````


I mentioned up above that the SDK has a built-in tool to help find cmdlets. By using `Find-MgGraphCommand` we can find PowerShell cmdlets for a given Graph endpoint. [I talked about how you can find Graph endpoints in a previous post.](https://www.modernendpoint.com/managed/Defining-a-script-and-finding-the-Microsoft-Graph-Queries/) If we want to find all of the available PowerShell cmdlets for `https://graph.microsoft.com/v1.0/DeviceManagement/ManagedDevices/` we can use the following command:


 ````powershell
 
#How to find command in PS
Find-MgGraphCommand -URI '/DeviceManagement/ManagedDevices/'
````


This returns all of the available cmdlets for this endpoint in both the `beta` and `v1.0` endpoints. Note that the response includes the command, the REST method, SDK module, and required permissions.
 

![Find Mg Graph Command](https://managedblog.github.io/managed/assets/images/22.05.08/03.FindMgGraphCommand.png){: .align-center}


In the above example we don't have the option to update a user. We can only get an existing user or create a new one. If you explore the Microsoft Graph documentation, you will find that we update a user by calling the above endpoint and appending the device's managed device Id.  By running `Find-MgGraphCommand -URI '/DeviceManagement/ManagedDevices/{id}'` we get the following response, which includes the `PATCH` method used to update the user.


![Find Mg Graph Commend Update Device](https://managedblog.github.io/managed/assets/images/22.05.08/04.FindMgGraphCommendUpdateDevice.png){: .align-center}
 

[In an earlier post in this series](https://www.modernendpoint.com/managed/Updating-device-management-name-in-PowerShell-with-Microsoft-Graph/), I demonstrated how to update the device management name on a user's Intune managed devices using Microsoft Graph. We can perform the same task in PowerShell with the following script:


````powershell
 
#Return all devices for the user labuser02@modernendpoint.xyz
$Devices = Get-MgDeviceManagementManagedDevice -Filter "UserPrincipalName eq 'labuser02@modernendpoint.xyz'" | Select-Object DeviceName,UserPrincipalName,Id,SerialNumber,ManagedDeviceName,OperatingSystem

#For each returned device, update the device management name
ForEach($Device in $Devices){
    $UPNPrefix = $Device.UserPrincipalName.split("@")[0]
    $OS = $Device.operatingSystem
    $Serial = $Device.SerialNumber
    $Id = $Device.Id
    $NewManagementName = "$($UPNPrefix)_$($OS)_$($Serial)"

    Update-MgDeviceManagementManagedDevice -ManagedDeviceId $Id -ManagedDeviceName $NewManagementName

}

#Wait 15 seconds and query the user's devices to check results
Start-Sleep -Seconds 15

Get-MgDeviceManagementManagedDevice -Filter "UserPrincipalName eq 'labuser02@modernendpoint.xyz'" | Select-Object DeviceName,UserPrincipalName,Id,SerialNumber,ManagedDeviceName,OperatingSystem
````


Running the script completes successfully and returns our updated device objects:


![Update Management Name Results](https://managedblog.github.io/managed/assets/images/22.05.08/05.UpdateManagementNameResults.png){: .align-center}
 

The PowerShell SDK is an effective tool for systems administrators. It's familiar. It's well documented. It can be used to perform a lot of the same tasks as we can do with direct API calls. It is an excellent option for an administrator who want to create automation workflows against Microsoft Graph, but is it the best option?
 

### Is there really a _better_ option?
----
 

There is an undeniable benefit to administrators using the Graph SDK. PowerShell is a familiar platform and is a skill that most of us are comfortable with. We aren't developers. We don't spend all day in code or working with APIs, and most of the Microsoft Graph documentation is targeted at developers who are already familiar with making REST calls. It takes time to learn a new skill, but sometimes the time invested pays off. 

PowerShell is a great tool. It lowers the barrier to entry for systems administrators to create solutions that range from UI-driven tools to back-end automations. It's integral to every system's engineer's job. Using direct REST API calls to Microsoft Graph isn't about replacing the SDK. It _is_ about adding another tool to your toolbox. You should use whatever tool works best for you and your scenario. I prefer making direct API calls.
 
Before I get into the reasons, I choose to make direct calls to Microsoft Graph, here's an example of how I work with Microsoft Graph. This script was described in detail [in a previous post](https://www.modernendpoint.com/managed/PowerShell-tips-for-accessing-Microsoft-Graph-in-PowerShell/). In this example, I am connecting to Azure Active Directory to get a bearer token using the MSAL.ps module. The `Invoke-MsGraphCall` function is used to call `Invoke-RestMethod`. All of the parameters needed for the API call are splatted in a hashtable before being passed to `Invoke-RestMethod`. 
 
I know this looks like a lot of information, but if we break it down, we can see that it's actually a very simple block of code. Almost all of it can be reused to make any call to Microsoft Graph. The `Invoke-MsGraphCall` function can be used to make almost any REST API call without making any changes. At the start of the script, we gather information needed to connect to Azure and return an access token. I use a version of this in every script that I use to interact with Graph.


````powershell
#Call Get-MSALToken to get access token. In production secrets should be passed in securely from a key vault
$authparams = @{
    ClientId    = '[Client Id]'
    TenantId    = '[Tenant Id]]'
    ClientSecret = ('[Client Secret]' | ConvertTo-SecureString -AsPlainText -Force )
}

$auth = Get-MsalToken @authParams
$AccessToken = $auth.AccessToken
````
 

The next section of the script defines the function `Invoke-MsGraphCall`. We pass our access token, URI, REST method, and a body (if required) into the function. These parameters are added to a splat hashtable that is used to pass parameters into `Invoke-RestMethod`. If the method being used requires a body, the body is then added to the hashtable. Finally, we make our API call and return the response to the script.


````powershell
 
#Invoke-MsGraphCall function is used to create hashtable with all required parameters for Invoke-RestMethod and make the API call
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

    #Make API call with Invoke-RestMethod
    $MSGraphResult = Invoke-RestMethod @graphSplatParams

    #Return status code variable to script
    Return $SCV, $MSGraphResult
}
 
````


The last block of code is where the work is being done. This is typically the only section that needs to be updated. We are making a `POST` request to the endpoint `https://graph.microsoft.com/beta/users`. `POST`, `PUT`, and `PATCH` require a body to be passed in the request.  A request body is essentially a list of instructions that tells the endpoint which record to update and what properties need to be added or changed. This request has a more complex body than most, but I think it demonstrates the concept well. 
 
When you use this script to make a call to Microsoft Graph, you need to update the URI and Method. If the request will require a body, you simply replace the hashtable with your own and you are ready to make the API call.

 
````powershell
 
#In this example we are creating a new user. The listed parameters are required for creating a new user with Graph. The same parameters are used when using the SDK
    $Body= @{ 
        "accountEnabled" = $True
        "displayName" = "Display Name"
        "mailNickname" = "DisplayName"
        "userPrincipalName" = "displayname@modernendpoint.xyz"
        "passwordProfile"  = @{
          "forceChangePasswordNextSignIn" = $True
          "password" = "NewPassword"
        
        }
    }
 
    #Create required variables. In this case we are passing in the method and URI. A POST request requires a body. We are converting the hashtable to JSON.
    $URI = "https://graph.microsoft.com/beta/users"
    $Method = "POST"
    $Body = $Body | ConvertTo-Json
    
    #Call Invoke-MsGraphCall
    $MSGraphCall = Invoke-MsGraphCall -AccessToken $AccessToken -URI $URI -Method $Method -Body $Body
````
 

### That is different… is it better?
----

When I first wrote this post, this section was essentially a manifesto in favor of using direct API calls instead of using the SDK. There are still a lot of reasons to prefer `Invoke-RestMethod` over the SDK, but after working with the SDK the differences aren’t as wide as I expected.

There are four subjective areas where I think direct API calls win. _In my opinion_, making direct API calls with `Invoke-RestMethod` is a better option because it offers more simplicity, stability, flexibility, and portability. 
 
_**Simplicity**_
The script I shared above requires one module - MSAL.ps. Invoke-RestMethod is a native cmdlet in PowerShell core. It works without installing any extra modules and can be used to make API calls to _any_ REST API that you have access to.  The PowerShell SDK consists of a primary module and 38 sub modules. You don't need to install all of them, but you do need to know which ones need to be installed for the tasks you are trying to complete. If you choose only to install needed modules you may have to install additional modules later to makes calls to other areas of the service. Direct calls with `Invoke-RestMethod` only require you to know the URI, which is something you probably need to know to find the right cmdlet in the SDK.

_**Stability**_
Technology is subject to change. That applies to Microsoft Graph, whether your make direct calls to the API or use the SDK. However, API endpoints are much less likely to change. They form the backbone of the tools Microsoft has built into Azure, and deprecating an endpoint can be service impacting. The PowerShell SDK is a tool that Microsoft provides and updates regularly. Cmdlets get added and sometimes they get removed. When changes are made to Graph they are not going to be reflected in the SDK until several development cycles later. By making direct API calls to Microsoft Graph you can take advantage of new endpoints sooner, and they are going to be less likely to change. 


_**Flexibility**_
Do you want to authenticate to Microsoft Graph with a client secret stored in Azure Key Vault? Do you want to make calls to Microsoft Graph from your favorite low-code tool? Does your workload require you to hit other APIs? 
 
_Are any of those true in your environment? Consider the following:_

App-only authentication with the SDK requires a certificate. Client secrets are not supported.
 
Most low-code tools don't have an option to integrate PowerShell directly. If you want to use Power Automate or Logic Apps, you will have to call Azure Automation or an Azure Function.
 
The Microsoft Graph SDK support Microsoft Graph. If you want to make calls to other APIs, you're going to have to make a direct API call to those services. 
 
 
_**Portability**_
Portability and flexibility are closely related. Both the skills and the actual REST API calls that are used when making direct calls to Graph are portable. The SDK is self-contained. Knowing how to use `Get-MgUser` isn't going to help you when running a flow in Power Automate. If you want to return all users, you're going to make an HTTP request to `GET https://graph.microsoft.com/v1.0/users`. Working with third party APIs in PowerShell is identical to calling Graph with Invoke-RestMethod. The skillset itself is portable, which can't be said for the PowerShell SDK.

### You know, that’s just like, your opinion, man…
----

The above arguments are largely subjective. You may disagree with my reasoning, or you may think that there are other benefits that outweigh the ones that I have listed above. In the original draft of this post, I thought I had a rock solid fifth argument to make on why making direct API calls was far superior. I was wrong. In my early testing, I had found that using `Invoke-RestMethod` appeared to be around 10-12% faster than running an identical script built with the SDK. 

However, there was a problem. I didn’t want to share a small data set and say that direct API calls were always faster. I ran my test multiple times. I tried different sizes of datasets. The more I ran it, the less consistent my results were. What I ultimately found was that neither option held a large advantage while testing. I ran tests on various days with different sized datasets. The various test occurred on different networks. There were a lot of variables, but the results weren't enough to declare a winner. If anything, the SDK may be marginally faster than _my_ script using `Invoke-RestMethod`. 

To test the efficiency of both options I created two sample PowerShell scripts. They both authenticate to Microsoft Graph, start a timer, import a csv with a list of users, create the users, and then stop the timer. There are ways to optimize both scripts. In this example I wanted to keep it relatively simple, so each is performing a simple `ForEach` loop to iterate through each user in the csv files.


![Performance Testing Scripts Side By Side](https://managedblog.github.io/managed/assets/images/22.05.08/06.PerformanceTestingScriptsSideBySide.png){: .align-center}
 

The list of users we are creating is nearly identical. The only difference between the two files is the numbers in the usernames. 
 

![Import Users C S V Files](https://managedblog.github.io/managed/assets/images/22.05.08/07.ImportUsersCSVFiles.png){: .align-center}


When we run both options with a split terminal, we can see the time each took to run. In this example, the SDK ran in 9.1 seconds. `Invoke-RestMethod` took 8.9 seconds:
 

![Results Screen Shot](https://managedblog.github.io/managed/assets/images/22.05.08/08.ResultsScreenShot.png){: .align-center}


In other tests, the SDK was marginally faster. Scaling up made the results even more inconsistent. As I mentioned above, I couldn’t get consistent enough data to decide which option was faster. This surprised me. I expected that the abstraction and complexity of the SDK would introduce a few extra steps that may slow it down. That doesn’t appear to be true. I am comfortable to admit I was wrong, and the baseline performance doesn’t seem to be a reason to choose one option over the other.


### Well, that didn’t exactly clear things up
----

So which option should you use?

I think the correct answer to that question is, like my arguments above, largely subjective. It depends on a few different items. First, which one are you most comfortable with? I think there’s a lot of benefit to learning how to make direct API calls. There are situations where it is the better option. Learning the skills to manipulate an API directly will help you to work with other systems in your environment. If you are working with low-code tools, you will need to call a connector to run a PowerShell script directly.

Whichever option you choose for your daily tasks, I recommend taking some time to learn the other tool as well. Explore `Invoke-RestMethod` and look for other APIs you can work with in your environment. Take the time to learn the PowerShell SDK and test both options side by side. There isn’t a clear answer as to whether one option is better. There are use case for both, so find the option that works best for your scenario.

----

Thank you for reading! I have a lot of fresh content to post soon from MMS. Keep following for more great posts on automating workloads, including some fun PowerApps that I shared with the community!


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
14. [Comparing Invoke RestMethod to the PowerShell SDK](https://www.modernendpoint.com/managed/Comparing-Invoke-RestMethod-to-the-PowerShell-SDK)
