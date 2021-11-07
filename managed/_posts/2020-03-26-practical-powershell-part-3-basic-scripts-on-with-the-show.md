---
title: "Do I belong at this table?"
excerpt: "At a recent event I was sitting at a table with several Microsoft team members and customers. We all had one thing in common – a love for Microsoft’s Modern Endpoint Management solutions."
header:
    overlay_image:
    teaser:
tags:
  - PowerShell
---

It has been a long time since my last post on Practical PowerShell for the IT pro. A lot of things have changed in the world. The last few weeks have been a bit of a whirlwind. I would apologize for not making this post sooner, but I think it’s good for us to acknowledge the need to unwind. Focusing on writing has been hard, but now that I have adjusted to this new normal, I am ready to get back into the groove. For many of us, our jobs are unlikely to return the the “old normal.” Covid-19 will prove to be a world changing event, and the impact will be felt for years. Many of us have had to shift our approach from incremental change to rapid workplace modernization, and we have been asked to do it virtually overnight.

As we modernize, we will find a greater need to rely on PowerShell for managing users, devices, and licenses in the cloud. While the Administrative consoles give us access to most functionality, some controls aren’t exposed in the UI and many batch jobs or repetitive tasks can be completed more efficiently through a script. I hope that this blog series acts as a jumping off point for administrators that want to get started with PowerShell for systems administration.

In the last Practical PowerShell post I walked you through how to find the cmdlets you need to return an Azure AD user object. To complete that task, we had to install the Azure AD module, connect to Azure AD, then run the Get-AzureADUser cmdlet to return the user object. We ran each of these cmdlets separately to accomplish our goal. We can now use those cmdlets to create a basic script.​

It's Showtime!
----

A script is essentially just a set of directions that tells the computer what to do to return a desired result. Computer scripts aren’t that different the scripts used to produce a play or a movie. If you’ve ever acted in a play you will be familiar with the basic set description, stage direction, and dialog layout:

____

(The scene opens on a group of IT Pros working feverishly away at their desks. The set is dark, and we can see each IT pros face lit by the faint glow of a computer monitor. SEAN, a slightly unkempt but ruggedly good-looking IT pro bursts in stage R.)

Sean crosses to center stage, the other IT Pros take note.

**SEAN:** Everyone! Pay attention! I’m going to tell you about the MODERN WORKPLACE! And maybe even PowerShell!

____

As riveting as this scene sounds, I’m not convinced that I’m ready to write a Tony-award winning play. The opening section sets the scene. It provides important details on how the stage is laid out and where our actors are on stage. This is like the declarations section of a script where important variables may be set, connections are made, and sessions are imported. Those items set the stage to continue running the script.

After that we have a basic stage direction. This tells our dashing and debonair hero exactly what to do after entering on stage right.  If we connected to Azure AD in the first step, we could then tell PowerShell to run the Get-AzureADUser cmdlet. If we had not connected to Azure AD in the first step the next step, the next cmdlet wouldn’t have worked.

Finally, our lead character says his first line. This is essentially returning a value based on the rest of the script. Reciting a line is essentially the actor’s equivalent of a “Write-Host” command. If the stage crew got the lighting wrong before the curtain came up, the actors were not in their places, and props weren’t where they were intended to be the scene would have broken down (trust me, I know from experience!). The rest of the show wouldn’t go as planned.

The same is true with a PowerShell script. Each element must happen in the right order to get the desired result. Generally, variables that will be used throughout the script are set at or near the beginning, connections to outside services need to be made before running their associated cmdlets, and you can’t return a value until you have performed the tasks needed to collect that data.

Act II: The Script you’ve been waiting for
----

Today we want to return the devices associated with an Azure AD User. Based on the lessons in our last post, we know that we can search, “Get Azure AD devices by user” to see what options we have.

![Bing Search](https://managedblog.github.io/managed/assets/images/legacy/PS0326/01-get-device-by-user.jpg){: .align-center}

When we do this, we see that there is a cmdlet for this exact task, Get-AzureADUserRegisteredDevice:​

![Bing Search Results](https://managedblog.github.io/managed/assets/images/legacy/PS0326/02-results.jpg){: .align-center}

​When we look at the Microsoft documentation for the cmdlet we can see that the -ObjectID switch can be used to specify a user. 

![MS Get-AzureADUserRegisteredDevice documentation](https://managedblog.github.io/managed/assets/images/legacy/PS0326/03-ms-page.jpg){: .align-center}

​Earlier versions of the MS Online PowerShell module required the guid from the user object. Fortunately, in the current version of the AzureAD module we can simply use the User Principal Name.

```powershell

Get-AzureADUserRegisteredDevice -objectID sean@modernendpoint.com

```

![CMDLet Results](https://managedblog.github.io/managed/assets/images/legacy/PS0326/04-get-azure-aduserregistereddevice.jpg){: .align-center}

Now we have all the cmdlets we need to connect to Azure Activity Directory, return a user, and return their active devices. From here we can build a script that would take a single user and return the information we need to know about that user and their devices.

We have already established that a script is simply a collection of directions that tells a computer exactly what we want it to do. It can be as simple as a single cmdlet or could stretch for thousands of lines. It doesn’t always make sense to write a script to complete a task. If you won’t need to repeat a task it probably makes more sense to run the individual cmdlets one time. Tasks that are often repeated or run on a schedule will usually merit writing a script. My benchmark is generally based on making a best guess as to whether writing a script will save me time in the long run. If it takes five minutes to run through individual cmdlets to return user licensing information, but an hour to write the script, it probably doesn’t make sense to write that script. If we need to return information on a user and their devices whenever someone leaves the company, we could be performing this task hundreds of times a year. In that case a script not only makes sense, but it will save countless hours and moves you one step closer to automating a repetitive task.

On with the show!
----

When we write scripts, we need to know what information we have to begin with and what we expect our script to do with that information. In this example we will assume that we receive a notification that a user is leaving our company. Human Resources sends us the user’s name and their email address, and we need to send back information about their account and devices that the user has registered.

When I write a script, the first step I take is to create a list of tasks that need to be completed. This is one example of writing pseudocode. It’s clean and helps me to organize my thoughts to build a script.

1. Input user’s email address
2. Connect to Azure AD
3. Get the user based on their email address
    * Return user properties
4. Get the list of devices registered to the user
    * Return name
    * Return device ID
    * Return Device type

Now that we have an outline of what the script should look like we can begin to build our script. I am going to use the script pane in the PowerShell ISE to begin to build my script. We want to be able to repeat this script for any user that leaves the company, and I don’t want to have to change multiple references to the user’s email every time I run it. To accomplish this task, I am going to pass the email address into a variable that can be reused in the Get-AzureADUser and Get-AzureUserRegisteredDevice cmdlets. In our environment a user’s primary email matches their User Principal Name (UPN), which is passed into the -ObjectID switch.
​
In the ISE I am going to create a variable ($UPN) and set it to my UPN. The green text on the first line is a comment that explains what I intend to do in the following section of code. It’s best practice to put comments in your script. This acts as a guide to help you remember what your script does and why you’re doing it in the way that you are. If you share the script with others it will help them to understand what each section of the script does. It’s not required but will make your life easier later. Create a comment in PowerShell by using the pound [\#] symbol at the start of a line. _(Helpful hint: When troubleshooting a script or making edits later you can comment out lines of a script to prevent them from running!)_

```powershell
#Set the User Principal Name to return information for
$UPN = "sean@modernendpoint.com"
```

The Connect-AzureAD cmdlet can be passed into the script with no further switches. In my next blog post I will explain the use of functions and we will explore other options to use with the Connect-AzureAD cmdlet. For the purpose of this post we are going to keep things simple.

```powershell
#Set the User Principal Name to return information for
$UPN = "sean@modernendpoint.com"

#Connect to Azure Active Directory
Connect-AzureAd
```

Next, we will get the user’s account information from Azure. HR wants to know a user’s country, whether they have assigned licenses, and if their account is still active. To accomplish this task, we want to return the DisplayName, Country, AccountEnabled, and AssignedLicenses properties. We will pipe the results of Get-AzureADUser to Select-Object and pass the property names we want to return. To make it easier to work with these later we will pass them to the $UserProperties variable.

```powershell
#Set the User Principal Name to return information for
$UPN = "sean@modernendpoint.com"

#Connect to Azure Active Directory
Connect-AzureAd

#Get user information from Azure AD.
$UserProperties = Get-AzureADuser -ObjectId $UPN | Select displayName,Country,AccountEnabled,AssignedLicenses
```

The next step in our script will be to get the user’s device information. To do that we will run the Get-AzureADUserRegisteredDevice. We want to know the device’s name, operating system, and OS version.

```powershell
#Set the User Principal Name to return information for
$UPN = "sean@modernendpoint.com"

#Connect to Azure Active Directory
Connect-AzureAd

#Get user information from Azure AD.
$UserProperties = Get-AzureADuser -ObjectId $UPN | Select displayName,Country,AccountEnabled,AssignedLicenses

#Get user device information
$Devices = Get-AzureADUserRegisteredDevice -ObjectId $UPN | Select DisplayName,DeviceOSType,DeviceOSVersion
```

The final step is to return the values from our Get- cmdlets. You can return a variable’s value to the host by simply calling the variable.

```powershell
#Set the User Principal Name to return information for
$UPN = "sean@modernendpoint.com"

#Connect to Azure Active Directory
Connect-AzureAd

#Get user information from Azure AD.
$UserProperties = Get-AzureADuser -ObjectId $UPN | Select displayName,Country,AccountEnabled,AssignedLicenses

#Get user device information
$Devices = Get-AzureADUserRegisteredDevice -ObjectId $UPN | Select DisplayName,DeviceOSType,DeviceOSVersion

#Return Values to Console Window
$UserProperties
$Devices
```

You can run a script in the ISE by clicking the Green Arrow on the command bar. You can run it from the script pane by browsing to the directory it is saved in and typing .\ followed by the script name.

![Running the script in the ISE](https://managedblog.github.io/managed/assets/images/legacy/PS0326/10-script-6_orig.jpg){: .align-center}

When we execute the script, we get the following output:

![Script output](https://managedblog.github.io/managed/assets/images/legacy/PS0326/11-output_orig.jpg){: .align-center}

We can see at the top the account that was used to connect to Azure. This tells us that our Connect-AzureAD cmdlet was successful in connecting. We then received our user properties and finally the device properties. This script works as it was intended, but there are several items that can be cleaned up. AssignedLicenses is a multi-value property, and the information it returned isn’t very clear. I can see that this user is licensed, but unless I know which licenses the SkuID field corresponds to the information isn’t very useful. Also, the user properties and device properties are in separate variables which means they are separate objects. This will make it more difficult to work with them in a more complex script or to return them in a more usable format.

In future blog posts I will show you how to work with objects and to write out information that is returned to a file. We will also work with multi-value properties and I will show you how to pass the SkuID into Get-AzureADSubscribedSku to return more information about a user’s licenses. Once again, the point of this series is to create a series of cumulative lessons to take an IT pro that is new to PowerShell from running basic cmdlets to building more complex scripts. It’s not meant to be an exhaustive resource, but hopefully it provides you the information you need to build your comfort level in PowerShell.