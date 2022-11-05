---
title: "Migrating AD Domain Joined Computer to Azure AD Cloud only join"
excerpt: "Over the years, a lot of people have been looking for a solution to migrate on-premises Active Directory joined devices to Azure Active Directory cloud-only join." 
tags:
  - Intune
  - PowerShell
  - Azure
  - Azure Active Directory Join
  - AAD Migration
  - Device Migration
  - OneDrive for Business
  - Domain join to AAD Join
  - Domain join to Azure join

---

_<small>Hello, and welcome back! Today I am sharing a tool to help you migrate a domain-joined device to a cloud-only, Azure AD joined state. Instead of my normal, in-depth step by step, this post will provide a high-level overview on how the tool works. **Please note:** This is not an officially supported solution. Your mileage may vary, and if you choose to use it, please test thoroughly before using in production.</small>_


![Header Image](https://managedblog.github.io/managed/assets/images/22.11.05/01.HeaderImage.jpg){: .align-center}
 

In [my last blog post](https://www.modernendpoint.com/managed/Dynamically-Update-Primary-Users-on-Intune-Managed-Devices/), I shared a tool that uses a proactive remediation and Azure Automation webhook to dynamically set and update the primary user on an Intune-enrolled device. I also promised to share another tool with the community, and that’s what today’s post will do. I normally approach these posts with an in-depth, step-by-step analysis of how the tool works. In today’s post, I am going to take a slightly different approach. Over the years, a lot of people have been looking for a solution to migrate on-premises Active Directory joined devices to Azure Active Directory cloud-only join. 

## First, some disclaimers

The only officially supported method of doing this is through a wipe and reload. The recommended and most common path for moving to Azure-AD only join is to hybrid join existing devices, while using Autopilot to join Azure AD for all new devices. I wanted to find a solution to allow an organization to migrate their devices to a cloud-only join state without using hybrid join or wipe their devices. I _highly_ recommend using one of the commonly supported methods to begin your cloud endpoint journey, but I also acknowledge that in some instances organizations require minimal downtime and end user impact. [Adam Nichols](https://twitter.com/mauvlan) talked about how this could be accomplished at MMS in May, but focused on how it could be done using Configuration Manager. He did share a proof of concept on how it could be done without Configuration Manager using a provisioning package and PowerShell. [His script is available on his blog](https://mauvtek.com/home/active-directory-join-to-azure-ad-join). I want to echo and amplify his statement, _**”This is not a Microsoft supported process today.”** I will not be supporting this tool, but I did want to share it with the community as an alternative method for you to test and consider in your environment. I may make some updates to the tool, but those updates will largely be done as skill builders for me personally, not intended to make this a production tool.



In most cases, people are asking for a migration path that will migrate a user’s profile. There are some third-party tools that claim to do that, but I personally believe migrating a user profile is the riskiest and most time-consuming part of the process. I have heard of mixed results when using tools that claim to migrate a user’s profile. To avoid any issues with profile migration, this solution turns on OneDrive known folder move and points it at the target tenant. There is an Intune policy in the target tenant that turns on known folder move, allowing content in a user’s Documents, Desktop, and Pictures folder to move seamlessly between the old profile and new profile. Other profile information is not moved, but you could consider using a tool to capture a list of installed printers and user applications and apply those post-migration. Remember, that on a domain-joined computer the profile is _directly_ tied to the Active Directory user object. This breaks that connection, so the profile _must_ change when the domain is no longer available.

With that being said, this tool is _only_ designed to migrate the device state. OneDrive Known Folder move is the best option for migrating user data. One advantage of this method is that the device is not wiped. If a user stored data outside of the supported known folders, that data will still be available on the device.

# The Tool

I created the original version of this tool to assist someone who needed to move devices that were in one of several different management states to a single, Azure Active Directory joined state. They had devices in two different domains. One of the domains was being completely deprecated. Approximately half of their devices were not joined to any directory. Users had accounts in two domains and logged into unmanaged devices with local user accounts. They had a third-party management tool with limited ability to push files to devices. Their environment posed multiple challenges. They wanted an option that would give them a single path to migrate all their devices to a single Azure Active directory tenant and preserve user data in the process. Since so many of their devices were largely unmanaged and had no onsite IT support, we needed a method that would be quick and relatively light touch.

Adam’s script provided an excellent framework. He uses a single script and a provisioning package. The script moves all the needed files to a folder in ProgramData and creates multiple _PostRun_ scripts which are triggered using the NextRun registry key. A local administrative account is created in the initial script. That script sets multiple registry values, including allowing the temporary admin user to automatically login and launch each successive script. The provisioning pack uses a bulk enrollment token to join Azure AD after the first logon. After the second logon the device escrows its BitLocker keys and autologin is disabled. After the user logs in, additional cleanup tasks are completed. I found this approach to be novel. I had never considered leveraging the NextRun key in this manner.

After reviewing Adam’s proof of concept, I found that there were a few areas that could be improved upon. Since we had to deliver multiple files to the computer anyway, I elected to remove the script transcription from the process. Additional error handling was added to account for potential failure points. Because we were relying on OneDrive Known Folder Move, we wanted to make sure OneDrive was installed, known folder move was enabled, and that the sync state was healthy. Ideally, we wanted the tool to run on user logon instead of interrupting their workday. It needed to be interactive so they would know when it was going to run and defer it if possible. After the migration, the end user isn’t an administrator, so the cleanup script needs to be able to run for a standard user. We also wanted to remove any local administrative users and cleanup any changes we made to allow the process to run.

The final product runs in two parts. The first step, `Prepare-DeviceMigration.ps1`, creates the folder `C:\ProgramData\AADMigration` and unzips the contents of `AADMigration.zip`. 


![A A D Migration Folder Created](https://managedblog.github.io/managed/assets/images/22.11.05/02.AADMigrationFolderCreated.jpg){: .align-center}
 

OneDrive is installed if it isn’t already on the machine and registry keys are set to enable known folder move. A new Event Source is created to monitor the status of OneDrive sync and two scheduled tasks are created. The first scheduled task, `AADM Get OneDrive Sync Status` calls a PowerShell script that uses [OneDriveLib.dll](https://github.com/rodneyviana/ODSyncService/tree/master/OneDriveLib) by Rodney Viana to check the current synchronization status. It writes an entry in the event log to record the status. The second scheduled task, `AADM Launch PSADT for interactive Migration`, calls `Launch-DeployApplication_schtask.ps1`. This script runs as `SYSTEM` and uses `ServiceUI.exe` from the Microsoft Deployment Toolkit to call `Deploy-Application.exe` and the [PowerShell App Deploy Toolkit](https://psappdeploytoolkit.com/). _(Please note, I didn’t create that script and somehow lost track of the source – if you know who wrote it, please let me know so I can credit them.)_


![Scheduled Task Created](https://managedblog.github.io/managed/assets/images/22.11.05/03.ScheduledTaskCreated.jpg){: .align-center}
 

Both scheduled tasks run on user login with a 1-minute delay. If OneDrive is in a healthy sync state, `Deploy-Application.exe` will run. The user will be prompted to continue with the deployment or defer. The message box and deferral settings are controlled by `Deploy-Application.ps1`, and for the purpose of this tool, can be set by updating the values in the `MigrationConfig.psd1` located in the `Scripts` folder.


![User Migration Prompt](https://managedblog.github.io/managed/assets/images/22.11.05/04.UserMigrationPrompt.jpg){: .align-center}
 

If the user clicks on the continue button, the migration process will begin.


![Migration In Progress](https://managedblog.github.io/managed/assets/images/22.11.05/05.MigrationInProgress.jpg){: .align-center}
 

The first script in the process is `Migrate-toAADJOnly.ps1`. The contents of this script were added to `Deploy-application.ps1`, so the script is never explicitly called. This script creates a local admin user named `MigrationInProgress` and sets up automatic login. The script checks to see if the `AAD Join.ppkg` package had attempted to install previously and removes it if it failed. BitLocker protection is suspended. Multiple registry keys are set to allow the process to run smoothly. `NextRun` is set to run the script `PostRunOnce.ps1`. Any existing directory or Intune join is removed and the computer reboots.

When the computer reboots, the user will initially be met with a login screen that shows the user `MigrationInProgress` is logging in.


![Migration In Progressuser Logging In](https://managedblog.github.io/managed/assets/images/22.11.05/06.MigrationInProgressuserLoggingIn.jpg){: .align-center}       
 

After the user login completes, PostRunOnce blocks user input and displays an image that states device migration is in progress. This message is displayed while the provisioning package, `AAD Join.ppkg`, is installed. The provisioning package uses a bulk enrollment token to join Azure Active Directory, and automatic Intune enrollment is configured. `NextRun` is set to run `PostRunOnce2.ps1`. Once this is completed, the computer restarts.


![Device Migration In Progress Image](https://managedblog.github.io/managed/assets/images/22.11.05/07.DeviceMigrationInProgressImage.jpg){: .align-center}
 

`PostRunOnce2` also blocks user input and displays the above message. The computer escrows its BitLocker keys to Azure Active directory. A scheduled task is created to run `PostRunOnce3.ps1` on user login, and automatic logon is disabled. The legal notice caption and text are set in the registry to display a message letting end users know the process has been completed.
After reboot, the legal notice displays.


![Migration Completed Legal Notice](https://managedblog.github.io/managed/assets/images/22.11.05/08.MigrationCompletedLegalNotice.jpg){: .align-center}
 

After clicking okay, the login screen prompts the user to log in with their work or school account.


![Work Or School Account Login Screen](https://managedblog.github.io/managed/assets/images/22.11.05/09.WorkOrSchoolAccountLoginScreen.jpg){: .align-center}
  

After user logon has completed, we can see the device has enrolled in Azure Active Directory and is no longer domain joined by running `dsregcmd.exe /status` from an administrative command prompt.


![Dsregcmd Azure A D Joined](https://managedblog.github.io/managed/assets/images/22.11.05/10.DsregcmdAzureADJoined.jpg){: .align-center}
 

If you are also using [my utility to automatically update the primary user](https://www.modernendpoint.com/managed/Dynamically-Update-Primary-Users-on-Intune-Managed-Devices/), you should see the Primary User change to the user who logged in to the device. Otherwise, devices enrolled with a bulk enrollment token will not have an assigned primary user and will be considered shared devices.


![Primary User Updated](https://managedblog.github.io/managed/assets/images/22.11.05/11.PrimaryUserUpdated.jpg){: .align-center}
 

# Setting up the solution

I know what you’re thinking, “Okay, that’s cool, but how do I use it?”

First, remember this _**is not**_ a supported solution, so if you want to use it in your environment, thoroughly vet it before deploying it to any production computers. Before using this tool, please consider all supported options, including hybrid Azure AD Join and performing a wipe and reload. No warranty is expressed or implied. Feel free to use or modify it, but _**any production use is done at your own risk**_ and should only be done after thorough testing.

## Prepare the solution directory

1.	[Download the migration tool from my GitHub](https://github.com/managedBlog/Managed_Blog/tree/main/AD%20to%20AAD%20Only%20Migration%20Tool/Beta%20-%20PS1%20Script%20based)
2.	[Download the PowerShell App Deploy Toolkit](https://github.com/PSAppDeployToolkit/PSAppDeployToolkit/releases) and place the contents in the “Toolkit” folder
3.	[Download the Microsoft Deployment Toolkit](https://www.microsoft.com/en-us/download/details.aspx?id=54259) and install it. Copy ServiceUI.exe from the C:\Program Files\Microsoft Deployment Toolkit\Templates\Distribution\Tools\x64 folder and place it in the Files directory
4.	[Download OneDriveLib.dll](https://github.com/rodneyviana/ODSyncService) and place it in the files directory
5.	[Create a provisioning package to do AAD bulk enrollment](https://mauvtek.com/home/azure-ad-join-provisioning-package) and place it in the Files directory
6.	Replace the Deploy-Application.ps1 in the toolkit folder with the copy that is located in the Scripts folder.
7.	Replace the banner image (`AppDeployToolkitBanner.png`) in Toolkit\AppDeployToolkit with your banner image.

## Update the MigrationConfig.psd1 file
1)	Update the values in AADMigration\Files\MigrationConfig.psd1
a.	UseOneDriveKFM – Set this to $True to install OneDrive and automatically enable Known Folder Move in the target tenant.
b.	TenantID – This it the tenant ID of the target tenant. It is required when using OneDrive Known Folder Move.
c.	DeferDeadline – This is used in Deploy-Application.ps1 to determine the deferral deadline for completing the migration.
d.	DeferTimes – This will allow a user to defer the migration a specified number of times. Please note, that you can only use one of these deferral options.
e.	StartBoundary – This value is used to determine when the scheduled task will start running to begin the migration. 
f.	TempUser = This is the name of the temporary user account that will be created to complete the migration. It will be created as a local admin and configured for automatic login.
g.	TempPass – This is the password that will be used for the TempUser account. The TempUser and Password will be created ONLY when the migration begins and be deleted at the end of the process
h.	DomainLeaveUser – This is not required. If the DomainLeaveUser is present, the utility will check for connectivity to a domain controller and try to gracefully leave the domain. The account needs permission to add and remove computers from the domain and delete computer objects. If it is not present, the utility will disable network adapters and use the local account to leave the domain, then re-enable the network adapter before rebooting. If a domain controller cannot be contacted, the device will need to be manually deleted from the domain.
i.	DomainLeavePass – This is required if the domain leave user is specified.
j.	ProvisioningPack – This is the name of the provisioning package used for bulk enrollment in Azure Active Directory.

## Determine your delivery method and update Prepare-DeviceMigration.ps1
The tool was designed to be delivered with two files – `Prepare-DeviceMigration.ps1` and `AADMigration.zip`. Create the zip file by adding the entire AADMigration folder to a compressed zip file. If your delivery mechanism supports it (for example, using group policy) you can place both files in the same directory and then run the PowerShell script.

If your tool does not support this delivery mechanism, you may need to find another way to deliver the files to the device. If needed, update Prepare-DeviceMigration.ps1 to work with your deployment method.

At this point, you should be ready to test the migration in your environment. 

# Thank you!

I hope this solution helps to demonstrate the art of what’s possible and find the best option for your environment. I absolutely recommend trying the officially supported options (hybrid join or wipe and reload) before considering an alternative solution. 

As I mentioned above, I may continue to develop this solution, but any updates will likely be to improve the delivery mechanism of the tool itself and modularize the solution. If you do have any questions, please feel to reach out to me. 

As always, if you enjoyed this post, please follow me for more great content in the future!
 
