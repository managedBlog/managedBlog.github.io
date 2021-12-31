---
title: "Connecting to Microsoft Graph with PowerShell"
excerpt: "We use a lot of different tools to automate tasks, but perhaps the most important tool for many of us is PowerShell. PowerShell is incredibly powerful, and we can leverage it to connect to the APIs. In this post, I will discuss how to connect to Microsoft Graph using PowerShell."
tags:
  - Microsoft Endpoint Manager
  - Intune
  - Microsoft Graph API
  - AdminService
  - PowerShell
---


_<small>Greetings and welcome back! This post is the second post in my series on Automating Endpoint Manager tasks using Graph API and the AdminService. My goal with this series is to provide a roadmap for Endpoint Management professionals to begin working with APIs, especially in script building and toolmaking. Most of the resources around the various APIs are focused on developers. I want to provide a resource that works for endpoint managers. In my first post, I covered the basics – understanding API basics and how to form a simple API call. In this post I am going to build on that knowledge by showing you how to connect to Graph API using PowerShell. In future posts I will continue to build on these concepts to provide a guide on building tools that leverage these APIs.</small>_

| ![Microsoft Graph and PowerShell](https://managedblog.github.io/managed/assets/images/21.12.01/vsCode.png) |


Connecting to Microsoft Graph with PowerShell
-------

In my last post, I explored the basics of working with REST APIs. It was the start of a series on how we, as endpoint managers, can use Microsoft Graph and the AdminService. This post will provide another building block on how we can utilize these tools in our environments. We use a lot of different tools to automate tasks, but perhaps the most important tool for many of us is PowerShell. PowerShell is incredibly powerful, and we can leverage it to connect to the APIs. In this post, I will discuss how to connect to Microsoft Graph using PowerShell.

Connecting to Graph requires two parts – authentication and authorization. Authentication is proving you are who you say you are, while authorization is granting permissions to do something. When you authenticate to Microsoft Graph the service returns an authorization token. When you make calls to Microsoft Graph you provide the authorization token in the form of a bearer token in the Authorization header. 

When we connect to Graph through PowerShell, we are actually connecting to an application we have registered in Azure Active Directory. To do this, we need to create an App Registration. An App Registration is essentially the underlying definition of an application. It includes the application information including API permissions, secrets, and redirect URIs that the application is built on. The application become a service principal that we connect to. We connect with the information stored in the App Registration.

In many cases you can connect to an existing well-known application. I prefer to create an application that I control. This allows me to customize the permissions on a per-app basis. A common mistake many people make when working with App Registrations is to over-provision their permissions. By creating app registrations that only have the permissions needed to perform a specific task, we limit the risk of over-provisioning and can practice the principle of least privilege.

### Creating the App Registration

Open Azure Active Directory. Click on “App Registrations” in the menu. 

![Open App registrations in Azure](https://managedblog.github.io/managed/assets/images/21.12.01/01.OpenAppRegistration.png){: .align-center}

To create a new app registration, click on + New registration.

![Create a new registration](https://managedblog.github.io/managed/assets/images/21.12.01/02.NewRegistration.png){: .align-center}

Give your application a name. Leave the defaults on Supported Account Types and Redirect URI. Click Register to create the application.

![Register a new application](https://managedblog.github.io/managed/assets/images/21.12.01/03.RegisterAnApplication.png){: .align-center}  

After registering the application, you will be redirected to the application registration’s overview page. This page includes an overview of the basic information, including the Client ID, Tenant ID, and a link to the application in Enterprise applications. Once the application has been created, we will need to configure the application for signing in as a public application. Click on the “Authentication” option on the menu.

![Application Overview Page](https://managedblog.github.io/managed/assets/images/21.12.01/04.AppOverview.png){: .align-center}

Click on “Add a platform” under platform configurations.

![Platform Configuration](https://managedblog.github.io/managed/assets/images/21.12.01/05.PlatformConfiguration.png){: .align-center}   

Select “Mobile and desktop applications” from the menu. 

![Select a platform](https://managedblog.github.io/managed/assets/images/21.12.01/06.SelectPlatform.png){: .align-center}

Click on the checkbox for `https://login.microsoftonline.com/common/oauth2/nativeclient`.  Click Configure. 

![Configure app redirect URLs](https://managedblog.github.io/managed/assets/images/21.12.01/07.ConfigureDesktopAndDevices.png){: .align-center}

Scroll down to the bottom of the page and set “Allow public client flows” to yes. Click save at the top of the page.

![Allow public client flows](https://managedblog.github.io/managed/assets/images/21.12.01/08.AllowPublicClientFlows.png){: .align-center}

The application registration has now been configured to allow us to connect to the application. The next step is to grant permissions to Microsoft Graph.

![Select API Permissions](https://managedblog.github.io/managed/assets/images/21.12.01/09.SelectAPIPermissions.png){: .align-center}   

By default, our application has access to read user objects in Microsoft Graph. We want to add additional permissions for managing users, devices, and the Intune service. Click Add a permission.

![Default permissions](https://managedblog.github.io/managed/assets/images/21.12.01/10.DefaultPermissions.png){: .align-center}

We can give applications access to several different APIs that are available in Azure. In this case, we only want to give access to Microsoft Graph. Click on Microsoft Graph at the top of the screen.

![Select API](https://managedblog.github.io/managed/assets/images/21.12.01/11.SelectAPIs.png){: .align-center}

You will be asked to grant Delegated or Application permissions. Delegated permissions are used to have the application act on behalf of a signed in user. Application permissions allow the application to act as back-end service. For most endpoint management workloads, we will want to act as a service. Click on Application Permissions.

![Select Permission type](https://managedblog.github.io/managed/assets/images/21.12.01/12.SelectApplicationPermissions.png){: .align-center}

Clicking on Application permissions shows us the various API permissions that are available in Microsoft Graph. (Spoiler alert: there are a lot!). We want to create an application for managing typical endpoint management workloads, including managing users and group memberships. We will give the following permissions:

```
DeviceManagementApps.ReadWrite.All
DeviceManagementConfiguration.ReadWrite.All
DeviceManagementManagedDevices.PrivilegedOperations.All
DeviceManagementManagedDevices.ReadWrite.All
DeviceManagementRBAC.ReadWrite.All
DeviceManagementServiceConfig.ReadWrite.All
Directory.Read.All 
Group.ReadWrite.All
User.ReadWrite.All
```

You can use the search box to search for the various permissions and check the box to grant the permissions to the API. Click Add Permissions to add the API permissions to the application.

![Request Permissions](https://managedblog.github.io/managed/assets/images/21.12.01/13.RequestAPIPermissions.png){: .align-center}   

Each of the permissions that we just added needs to be consented to. It will appear on the configured permissions list with the status “Not granted for…”

![Added permissions](https://managedblog.github.io/managed/assets/images/21.12.01/14.ConfiguredPermissions.png){: .align-center}     

Click “Grant admin consent for…” to grant consent to the application. When prompted, click Yes to grant admin consent

![Grant consent](https://managedblog.github.io/managed/assets/images/21.12.01/15.GrantAdminConsent.png){: .align-center}

The permissions now show that consent has been granted for our tenant.

![Consent granted](https://managedblog.github.io/managed/assets/images/21.12.01/16.ConsentGranted.png){: .align-center}

At this point the application is configured to allow interactive authentication. Interactive authentication is a great solution for scripts that we will be running manually. To authenticate to Microsoft Graph we will connect using the Microsoft Authentication Library (MSAL). Using the MSAL module is the easiest way to authenticate to Microsoft Graph in different contexts. Install the module by entering `Install-Module MSAL.ps` in an administrative PowerShell Window.

![Install MSAL PowerShell Module](https://managedblog.github.io/managed/assets/images/21.12.01/17.InstallMOdule.png){: .align-center}

Once the module is installed, we are going to request an authorization token using the Get-MsalToken cmdlet. We will pass in our Tenant ID and the Client ID of the application. This will connect to the application, collect the registration information (including the API permissions) and build the Authentication Token. The Interactive switch is used to prompt for user authentication. The Client ID can be found on the Overview page of our Application Registration. The Tenant ID can be the tenant ID, Primary domain, or a verified domain name.

```powershell
$authParams = @{
    ClientId    = '01076ec2-ceb5-4662-82d1-a8e1f1b1fb50'
    TenantId    = 'modernendpoint.xyz'
    Interactive = $true
}
$auth = Get-MsalToken @authParams
$auth
```

When prompted, enter your credentials to sign in. 

![Sign in to Azure](https://managedblog.github.io/managed/assets/images/21.12.01/18.EnterCredentials.png){: .align-center}

After authenticating, you will receive a response. The $Auth in the code snippet above will return the token. The AccessToken is the value that we will need to pass into our API calls to interact with the Microsoft Graph.

![MSAL Token returned](https://managedblog.github.io/managed/assets/images/21.12.01/19.MSALTokenReturned.png){: .align-center}

We can test our connection to GraphAPI by making an API call to return our user object. We will use the same API call from the first post in this series – `GET https://graph.microsoft.com/v1.0/me`.

```powershell
$AccessToken = $Auth.AccessToken

#graph
$graphGetParams = @{
    Headers     = @{
        "Content-Type"  = "application/json"
        "Authorization" = "Bearer $($AccessToken)"
    }
    Method      = "GET"
    ErrorAction = "SilentlyContinue"
    URI = "https://graph.microsoft.com/v1.0/me"
}

Invoke-RestMethod @graphGetParams 
```

This API call will return the user object that we originally authenticated with.

![Returned user object](https://managedblog.github.io/managed/assets/images/21.12.01/20.ReturneduserObject.png){: .align-center}

This works great for interactive authentication, but what if we want to run a script as part of an automation workflow or on a schedule? We wouldn’t be able to authenticate interactively. We could always hard code a username and password, but that’s not a good idea (for obvious reasons). The other (better) option is to create a client secret in our application registration and pass it in from a secure password vault. In a future blog post I will cover how to store credentials and client secrets in Azure Key Vault, but in this post, I want to cover how to create and use a client secret to authenticate to Microsoft Graph.

In the App Registration, click on Certificates & Secrets.

![Select Certificates and Secrets](https://managedblog.github.io/managed/assets/images/21.12.01/21.CertsAndSecrets.png){: .align-center}

Click New Client Secret.

![New Client Secret](https://managedblog.github.io/managed/assets/images/21.12.01/22.NewClientSecret.jpg){: .align-center}

Name your client secret and select an expiration period. Click Add to create the secret.

![Add a client secret](https://managedblog.github.io/managed/assets/images/21.12.01/23.AddClientSecret.png){: .align-center}

Copy the client secret Value and save it in your password manager. The client secret is ONLY viewable when it is first created. If you lose the secret, you will need to create a new one.

![Client secret in app](https://managedblog.github.io/managed/assets/images/21.12.01/24.ClientSecret.png){: .align-center}

Our commands to get the authorization token are almost identical. In this case we are going to replace “Interactive = $true” with our Client Secret. When passing in a credential, the password has to be passed as a secure string. In this example we pipe the secret to the `ConvertTo-SecureString`.

 I know my Client Secret is visible here – don’t worry, this application registration was removed before this post went live! If you’re new to App Registrations, be sure to treat your Client Secrets just like a password. They should be saved in a secure fashion and never be shared. In later blog posts I will be storing my keys in Azure Key Vault. In that post, I will show you both how to save the keys and how to use them.

```powershell
$authparams = @{
    ClientId    = '01076ec2-ceb5-4662-82d1-a8e1f1b1fb50'
    TenantId    = 'modernendpoint.xyz'
    ClientSecret = ('nV87Q~sQFYUd7Y7WJeANkWkfD5aEC9H_W~SRP' | ConvertTo-SecureString -AsPlainText -Force )
}
$auth = Get-MsalToken @authParams
$auth
```

This returns the same result as we received above when authenticating interactively.

![Final token returned](https://managedblog.github.io/managed/assets/images/21.12.01/25.FinalResults.png){: .align-center}

By creating and connecting with the client secret we can connect programmatically to Microsoft Graph. This allows us to automate the connection and will be very helpful when creating tools to manage different workloads. Throughout the rest of this series, we will be using the Microsoft Authentication Library, particularly the client secret above to connect to Microsoft Graph. It is extremely useful for accessing resources from several different authentication tools.


____
Follow the full series below:

1. [Everything I wanted to know about APIs but was afraid to ask](https://www.modernendpoint.com/managed/everything-i-wanted-to-know-about-apis-but-was-afraid-to-ask/)
2. [Connecting to Microsoft Graph with PowerShell](https://www.modernendpoint.com/managed/connecting-to-microsoft-graph-with-powershell/)
3. [Troubleshooting Microsoft Graph App Registration Errors](https://www.modernendpoint.com/managed/troubleshooting-microsoft-graph-app-registration-errors/)
4. [Defining a script and finding the Microsoft Graph Calls](https://www.modernendpoint.com/managed/Defining-a-script-and-finding-the-Microsoft-Graph-Queries/)
5. [Splatting with Invoke-RestMethod in PowerShell](https://www.modernendpoint.com/managed/PowerShell-tips-for-accessing-Microsoft-Graph-in-PowerShell/)