---
title: "Practical PowerShell: Part 4 - Function Junction"
excerpt: "It’s taking everything in me right now to not break into School House Rock’s “Conjunction Junction,” but unfortunately this post is about functions, not conjunctions."
header:
    overlay_image:
    teaser:
tags:
  - PowerShell
---  

<small>_This post is part 4 in an open ended series on PowerShell for IT Pros. It is intended to be a framework to learn from, not an exhaustive study guide. Consider it a survey in what's possible, not a master's class in all things PowerShell. [You can find the first post in the series here](https://www.modernendpoint.com/managed/practical-powershell-where-do-i-start)._</small>

____

It’s taking everything in me right now to not break into School House Rock’s “Conjunction Junction,” but unfortunately this post is about functions, not conjunctions.

A function in PowerShell is a block of code that you can call from within a script. Parameters can be passed into a function, and it can be called any number of times. I typically write functions for one of two different purposes: either I need to run the same block of code multiple times in the same script, or I plan on reusing the code block in multiple scripts.

Functions are essentially scripts that are included as part of a larger script. You will often hear experienced scripters talking about building a toolbox. For many of us, that is a collection of scripts and functions that we reuse regularly. For example, I have several different functions that I use as source material for other scripting project. I have a logging function, a function that writes an exit code to the registry, and functions to build PowerShell objects. Some of these, like the logging and exit code functions, can be plugged into a script with almost no changes. The PS Object function usually needs to be customized based on the script I am work on. Whether a function needs to be edited or can be plugged directly into another script – they all save time and assist with common tasks.

A function can be as simple as a single cmdlet, or as complex as many scripts. Generally, I like to keep my functions simple. If it needs dozens of lines, it may be best broken out into small functions. Conversely, if a task can be handled in one or two cmdlets, it may be harder to justify handling it in a function. In this post we are going to explore a relatively simple function, building off the Connect-AzureAD cmdlet from my previous post.
​
Does it function?
----

The Connect-AzureAD cmdlet is an elegant cmdlet. It works well. When you run the cmdlet, you are given a familiar Microsoft account login screen. You can easily enter your credentials and continue without having to call another function. It’s clean, simple, and there’s not much need to create a function if all we want to do is connect to Azure Active Directory. However, this basic function could be used as the building block for additional functionality down the road.

We create a function in PowerShell by calling the keyword, “Function,” followed by naming the function, and then adding the contents of the function inside a set of curly braces {}. A function should be named using a verb-noun pair (just like cmdlets). By clearly naming a function you will make it easier to remember the name of your functions, make your script more readable, and it will make it easier to reuse later.
​
If we wanted to create a basic function to connect to Azure Active Directory, it may look something like this:

```powershell
Function Connect-AzureWithCredentials {

	$Creds = Get-Credential
    Connect-AzureAD -Credential $creds
}
```

I could later call this function in a script by calling it by name, “Connect-AzureWithCredentials.” The get-Credential cmdlet prompts for credentials using a standard Windows credential dialog. Those credentials can then be passed into other cmdlets using a variable. 

![Get-Credential prompt](https://managedblog.github.io/managed/assets/images/legacy/PS0409/02-credential-screen_orig.jpg){: .align-center}

I know what you’re thinking. If the Connect-AzureAD cmdlet calls a login screen, why would we need a separate cmdlet to get credentials? This isn’t a very practical example on its own, but it becomes more useful down the road. If you need to connect to multiple Microsoft online services – for example Azure, Exchange Online, and SharePoint Online, you could connect to them with the same set of credentials in a single function.

What if we could pass in a credential without having to enter it each time? There are several ways of accomplishing this. None of them are completely secure, but there are a few options. For this example, I am going to use the Windows Credential manager to access a stored credential. This isn’t native PowerShell functionality, but can easily be accomplished by installing the CredentialManager PowerShell module. _(Note: there are people who will tell you that Credential Manager isn’t 100% secure. They’re right, but practically speaking if you practice good security hygiene it is REASONABLY secure – but you should take the time to understand the security implications for yourself)_.
​
Install the module by running Install-Module CredentialManager from an Administrative PowerShell prompt.

```powershell
Install-Module CredentialManager
```

​Once the module has been installed you can store a new credential using the New-StoredCredential cmdlet. When creating a stored credential use the -Target switch to name the credential, -UserName to assign the user, and -Password to store the password.

```powershell
#Create new stored credential
New-StoredCredential -Target AzureFunction -UserName Sean@GetYourOwnTenant.com -Password NotTODAY!
```

After running the cmdlet to store the credential, PowerShell will return the credential object. Here’s where we can clearly see the issue many people have with storing credentials in Credential Manager – the password is viewable by any process that is running elevated in our Windows session:

![Returned credential object](https://managedblog.github.io/managed/assets/images/legacy/PS0409/05-credential-object.jpg){: .align-center}

There are other ways we could further protect this credential, but I will save those for a future blog post. Since the primary goal of this post is to talk about building useful functions, I don’t want to get lost in the weeds on encrypting and decrypting strings in PowerShell.
​
Once we have our credential stored, we can use it our function. We can store the results of the Get-StoredCredential cmdlet in a variable. When we return the variable, it will return the username and password. When returning the credential in this way, the password is returned as a secure string that we can pass into the Connect-AzureAD cmdlet.

![Retrieving the stored credential](https://managedblog.github.io/managed/assets/images/legacy/PS0409/06-stored-credential.jpg){: .align-center}

Our updated function now looks like this:

```powershell
#Function to connect to Azure AD with stored credentials
Function Connect-AzureWithCredentials {

	$Creds = Get-StoredCredential -Target AzureFunction
    Connect-AzureAD -Credential $creds
}
```

We can add this function to our original script. In PowerShell, functions need to be added to the script before they can be called. I add any functions I plan on using to the start of a script. In this case, we call our function in line 15. Note that to call the function we simply call the function’s name.  

```powershell
#Function to connect to Azure AD with stored credentials
Function Connect-AzureWithCredentials {

	$Creds = Get-StoredCredential -Target AzureFunction
    Connect-AzureAD -Credential $creds
}


#Set the User Principal Name to return information for
$UPN = "sean@modernendpoint.com"

#Connect to Azure Active Directory using function
Connect-AzureWithCredentials

#Get user information from Azure AD.
$UserProperties = Get-AzureADuser -ObjectId $UPN | Select displayName,Country,AccountEnabled,AssignedLicenses

#Get user device information
$Devices = Get-AzureADUserRegisteredDevice -ObjectId $UPN | Select DisplayName,DeviceOSType,DeviceOSVersion

#Return Values to Console Window
$UserProperties
$Devices
```

​When we run the script, we can see that we got the same results as we did in the last blog post, only in this instance we weren’t prompted to enter our credentials. The function allowed us to run the script faster and, in this case, with minimal interaction. 

![Script results](https://managedblog.github.io/managed/assets/images/legacy/PS0409/09-results_orig.jpg){: .align-center}

Earlier I said that a function should serve one of two purposes, it should either include code that we need to repeatedly call in a script, or it should be portable and easily reused in other scripts. In this case, our script meets the second requirement. I could easily move this into other scripts. Its purpose is one of utility and convenience. It’s not overly complicated, and it serves a clearly defined purpose.
​
In my next post, I will explore using multi-valued properties. We will create a more complex function to extract the license SKUs and match those to a license and return the license plan information. 
