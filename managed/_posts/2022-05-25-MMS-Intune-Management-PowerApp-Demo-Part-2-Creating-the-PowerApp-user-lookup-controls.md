---
title: "MMS Intune Management PowerApp Demo Part 2: Creating the PowerApp user lookup controls"
excerpt: "This is a relatively simple app, but I will try to capture any of the details you may need to build your own copy. This app is designed to be a jumping off point for you to explore and understand the art of the possible. Use this tutorial to learn how to work Power Apps, and then use the app you build as a platform to build your own Intune device management mobile app."
header:
    og_image: "https://managedblog.github.io/managed/assets/images/22.05.25/23.txtUserValueSet.png"
tags:
  - Microsoft Endpoint Manager
  - Intune
  - Microsoft Graph API
  - AdminService
  - PowerShell
  - Azure
  - Azure Automation
  - Power Automate
  - Power Apps
---

Hello again! It feels like it was just yesterday when I last posted a new blog post. Wait. It was.

If you’ve been following along, you know that I have been writing a series on automating endpoint management tasks with Microsoft Graph and the Configuration Manager AdminService. After MMS I shared a [demo app that Doug Wilson (@manageDoug) and I built](https://www.modernendpoint.com/managed/Install-MMS-Demo-PowerApps-for-Intune/) during our session on building tools with Power Apps and Power Automate.  In [my last post](https://www.modernendpoint.com/managed/MMS-Intune-Management-PowerApp-Demo-Part-1-Creating-the-PowerAutomate-flows/) I shared step by step instructions on building the underlying Power Automate flows used in that app. In today’s post I will start walking through how to build the app itself. 

This is a relatively simple app, but I will try to capture any of the details you may need to build your own copy. This app is designed to be a jumping off point for you to explore and understand the art of the possible. Use this tutorial to learn how to work Power Apps, and then use the app you build as a platform to build your own Intune device management mobile app. 


### Enough chatter, let’s build something…
----

I have already introduced this app about fourteen different ways, and while I’m not known for brevity – let’s jump right in. 

In your browser, go to [Power Apps](https://powerapps.microsoft.com/). Click Start from Blank App.

 
![Start From Blank App](https://managedblog.github.io/managed/assets/images/22.05.25/01.StartFromBlankApp.png){: .align-center}


Click create on the blank canvas app card.

 
![Create Blank Canvas App](https://managedblog.github.io/managed/assets/images/22.05.25/02.CreateBlankCanvasApp.png){: .align-center}


Set the format to phone and name your app. In this example I am naming the app MMS-DemoApp.

 
![Name App Select Format Click Create](https://managedblog.github.io/managed/assets/images/22.05.25/03.NameAppSelectFormatClickCreate.png){: .align-center}

 
The PowerApp will open to a blank canvas. The first item we will add to the app is a screen with an Office 365 people picker. This will allow us to search for a user and populate a text box with their User Principal Name. There are several screen templates available. We will use a template and edit it to meet our needs. Click “New Screen” and select People.
 

![Add People Screen](https://managedblog.github.io/managed/assets/images/22.05.25/04.AddPeopleScreen.png){: .align-center}

 
The default people screen is designed to select multiple users and add them to a list. It also displays a message and icon if no users have been selected. We don’t need those elements in this app, so we will remove those objects from the screen.
  

![User Selection Screen](https://managedblog.github.io/managed/assets/images/22.05.25/05.UserSelectionScreen.png){: .align-center}


On the left navigation menu click on the ellipses next to `PeopleAddedGallery`. Click on delete to remove the menu.
  
 
![Delete People Added Gallery](https://managedblog.github.io/managed/assets/images/22.05.25/06.DeletePeopleAddedGallery.png){: .align-center}


Select the lblEmptyState and IconEmptyState. Delete the objects from the screen.
 

![Delete Label And Icon](https://managedblog.github.io/managed/assets/images/22.05.25/07.DeleteLabelAndIcon.png){: .align-center}


We now have a screen that shows only a search box. There is also a user browse gallery that is not currently visible.
 

![App Ready To Build](https://managedblog.github.io/managed/assets/images/22.05.25/08.AppReadyToBuild.png){: .align-center}


Click on the user browse gallery. To the left of the formula bar, select “Items” from the list of properties. The gallery has a formula in it that uses the Office 365 Users connector. The complete formula is `If(!IsBlank(Trim(TextSearchBox1.Text)), Office365Users.SearchUser({searchTerm: Trim(TextSearchBox1.Text), top: 15}))`.

If we break down the formula, we can see that it is a simple `if` statement. An `if` statement is done in Power Apps using a function that follows the format `if(condition, do something)`. 

The condition in this statement is `!IsBlank(Trim(TextSearchBox1.Text))`, which means, “if ‘TextSearchBox1 is blank’ is false.” 

If the text box is not empty, it will call the Office 365 Users connector and perform the `SearchUser` action. It will search for the text in textSearchBox1 and return the top 15 results.

 
![User Browse Gallery Items Formula](https://managedblog.github.io/managed/assets/images/22.05.25/09.UserBrowseGalleryItemsFormula.png){: .align-center}


If you want to test the app you are building, you can hit the play button in the upper right corner of the app. This will launch the app in PowerApps and allow you to test the current state. 
 

![Preview The App](https://managedblog.github.io/managed/assets/images/22.05.25/10.PreviewTheApp.png){: .align-center}


Search for a user with the app. You should see a list of users appear and be able to select a user. Click the `X` in the upper right corner to return to the canvas app 
 

![Searching For A User](https://managedblog.github.io/managed/assets/images/22.05.25/11.SearchingForAUser.png){: .align-center}


The canvas app will now show the data that was returned from preview mode. We can now see the user browse gallery and the various elements that were returned. The `OnSelect` property will not show a script in the formula bar. We will add a few more elements to the app, and then edit this script to control what happens when we click on the `Title` value in the userBrowseGallery.
 

![Title On Select Property](https://managedblog.github.io/managed/assets/images/22.05.25/12.TitleOnSelectProperty.png){: .align-center}


Click the `Text` button on the ribbon to add a new text-based element to your app. Select label to add a new label.

 
![Select Label](https://managedblog.github.io/managed/assets/images/22.05.25/13.SelectLabel.png){: .align-center}


You can rename the element by clicking on the name in the navigation menu on the left side of the screen. This label will be used to label a text box that will show the user returned from the user search function we added above. Rename the label to “lblSelectedUser.”
 

![Rename And Position Label](https://managedblog.github.io/managed/assets/images/22.05.25/14.RenameAndPositionLabel.png){: .align-center}


We can change the appearance and function of an element in our app by editing the properties on the right side of the screen. In this case, we want to adjust the side of the label field so it will fit with other elements in our app. Change the `Height` value on the `Size` property to 35.

 
![Set Size Of Label](https://managedblog.github.io/managed/assets/images/22.05.25/15.SetSizeOfLabel.png){: .align-center}


This label will be used as a visual cue for the text box that will display the selected user. Change the text property to “Selected User.” Set the font size to 12.

 
![Set Text Properties On Label](https://managedblog.github.io/managed/assets/images/22.05.25/16.SetTextPropertiesOnLabel.png){: .align-center}


Click the Text button on the ribbon and select Text input. This will add a textbox to our canvas. A text box will allow a user to enter a text value. In our app, a user could manually enter a value, or we can select a user from the userBrowseGallery and have it populate the text box.
 

![Add Text Input Control](https://managedblog.github.io/managed/assets/images/22.05.25/17.AddTextInputControl.png){: .align-center}


Position the text box below the selected user label. Power Apps will snap the text box into position below the label.


![Position Text Box](https://managedblog.github.io/managed/assets/images/22.05.25/18.PositionTextBox.png){: .align-center}


Rename the text box to txtUser.

 
![Rename Text Box](https://managedblog.github.io/managed/assets/images/22.05.25/19.RenameTextBox.png){: .align-center}


The textbox needs to update when we select a value from the userBrowseGallery. To do this, we will set the default value of this field to a variable that will be updated when we select a user from the search results. Select the `Default` property of `txtUser` and set it to `_VarSelectedUser`.

 
![Set Default Property To Variable](https://managedblog.github.io/managed/assets/images/22.05.25/20.SetDefaultPropertyToVariable.png){: .align-center}


Controls in PowerApps will appear in the order they show up in the tree view. If `txtUser` appears above `userBrowseGallery` in the tree view, it will always appear on top of the gallery when the search function is used. To prevent this from happening we can move it behind the gallery by right-clicking on the object on the tree view. Click Reorder and select “Send to back” to make sure it appears behind the gallery.

 
![Send Txt User To Back](https://managedblog.github.io/managed/assets/images/22.05.25/21.SendTxtUserToBack.png){: .align-center}


Click on the `Title` object in `userBrowseGallery`. Select the `OnSelect` property to view the script in the formula bar. 

We are going to update the `OnSelect` property by adding an additional action to the script. First, add a semi-colon `;` to the end of the last line. Add an additional to the line to the script and enter ` Set(_VarSelectedUser, _selectedUser.UserPrincipalName)`.

All of the actions included in the `Concurrent` action will be performed concurrently. In this case the variable `_selectedUser` is being set to the value `ThisItem`, which is the selected object from the gallery. The value of `TestSearchBox1` is being cleared, and finally, the selected item is being added to a collection called `MyPeople`. _(Note: I will explore galleries in more detail in my next post.)_

Finally, we are going to set the value of the variable `_VarSelectedUser` to the user principal name of the user we selected from the search box. We can set variables in PowerApps using the `Set` function. 

The `Set` function uses the format `Set(variable, value)`. In this instance, we are setting the variable `_varSelectedUser` to the `UserPrincipalName` property of the variable `_selectedUser`.


![Update Title On Select Formula Script](https://managedblog.github.io/managed/assets/images/22.05.25/22.UpdateTitleOnSelectFormulaScript.png){: .align-center}


Test the app by clicking the preview button and searching for a user. After selecting the user, the value of `txtUser` should be updated to show the correct user principal name.


![txt User Value Set](https://managedblog.github.io/managed/assets/images/22.05.25/23.txtUserValueSet.png){: .align-center}


### And now we wait…
----

To keep this post from getting obnoxiously long, I have broken it down into two separate posts. Keep an eye out here for the exciting conclusion! In my next post I will add two buttons to the app and a gallery object. We will use the two Power Automate flows from my [last post](https://www.modernendpoint.com/managed/MMS-Intune-Management-PowerApp-Demo-Part-1-Creating-the-PowerAutomate-flows/) to return a user’s devices and retire a device from Intune.

Thank you for reading!


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
15. [MMS Intune Management PowerApp Demo Part 1: Creating the PowerAutomate flows](https://www.modernendpoint.com/managed/MMS-Intune-Management-PowerApp-Demo-Part-1-Creating-the-PowerAutomate-flows)
16. [MMS Intune Management PowerApp Demo Part 2: Creating the PowerApp user lookup controls](https://www.modernendpoint.com/managed/MMS-Intune-Management-PowerApp-Demo-Part-2-Creating-the-PowerApp-user-lookup-controls)



