---
title: "MMS Intune Management PowerApp Demo Part 3: Adding the buttons, gallery, and completing the app"
excerpt: "In today’s post I will complete the app by adding a gallery and two buttons.  Those buttons will call the Power Automate workflows that call Microsoft Graph and will return and retire devices from Intune."
header:
    og_image: "https://managedblog.github.io/managed/assets/images/22.06.09/19.SelectDeviceAndClickRetire.png"
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

Hello and welcome back! I apologize for the delay in posting this blog, which will complete the series on building the demo Intune management Power App that Doug Wilson and I built during our session at MMS. The last several months have been extremely busy. I finally had a chance to take a break and go on vacation. I planned on completing the series before I left, but rather than rushing to complete this post I decided to relax and unplug. Mental wellbeing is more important than timely blog posts, so I decided to step away from the computer and begin my vacation a few hours earlier! 

I have already introduced the Power App in a [previous blog post and created the underlying Power Automate flows](https://www.modernendpoint.com/managed/MMS-Intune-Management-PowerApp-Demo-Part-1-Creating-the-PowerAutomate-flows/). In my [last post I created the app and added several controls to it](https://www.modernendpoint.com/managed/MMS-Intune-Management-PowerApp-Demo-Part-2-Creating-the-PowerApp-user-lookup-controls/). In today’s post I will complete the app by adding a gallery and two buttons.  Those buttons will call the Power Automate workflows that call Microsoft Graph and will return and retire devices from Intune.


### Let’s do this!
----

We’ve already added a user lookup tool and a text box to display user we select. Next, we will create two buttons. The first one will call the Get User Devices flow. The second one will call the Retire Device flow. Both flows will return a response to our Power App.

Click on the “Button” icon on the ribbon to add our first button.


![Select Button](https://managedblog.github.io/managed/assets/images/22.06.09/01.SelectButton.png){: .align-center}
 

We can rename the button by clicking on the button on the tree view on the left side of the canvas. Rename the button, in this case, I renamed the button to `btnGetDevices`. We will use the properties menu to change the button’s appearance. Change the `text` property to “Get Mobile Devices.” We can adjust the width of the button to better display the text. Set the `width` to 350 and the `height` to 70.


![Set Button Properties](https://managedblog.github.io/managed/assets/images/22.06.09/02.SetButtonProperties.png){: .align-center}


Click on the button it to position below the selected user text box. If you drag the button across the screen, it should try to snap to the grid created by other elements. Drag it toward the center and let it snap to the center of the canvas below the selected user text box.


![Position Get Devices Button](https://managedblog.github.io/managed/assets/images/22.06.09/03.PositionGetDevicesButton.png){: .align-center}


Copy the button and paste it back into the canvas. Rename the button and change the `text` property to “Retire Device”. _(I know that’s not what the screenshot says, but it’s a more accurate description of the action we are taking!)_ Drag the button to the bottom of the screen in line with the first button. It should snap to the correct vertical position.


![Position Retire Device Button](https://managedblog.github.io/managed/assets/images/22.06.09/04.PositionRetireDeviceButton.png){: .align-center}

 
The first button will call a Power Automate flow that will return a list of our selected user’s devices. When we return the devices, we want to display the device name and managed device ID. We can do that with a gallery control. Click on the “Gallery” button on the ribbon. This will open a menu with several different gallery options to choose from. For this example, select “Vertical.”


![Select Vertical Gallery](https://managedblog.github.io/managed/assets/images/22.06.09/05.SelectVerticalGallery.png){: .align-center}


In the gallery control’s properties, select Layout. Change the layout to “Title and Subtitle.” This will remove the preview image from the screen. The gallery will now show elements for the title, subtitle, and an arrow icon in the tree view.


![Set Gallery Layout](https://managedblog.github.io/managed/assets/images/22.06.09/06.SetGalleryLayout.png){: .align-center}
 

The gallery will appear with sample text. All the required controls have now been added to the canvas. Next, we will import the Power Automate flows that will be used to make calls to Microsoft Graph when we click on our buttons.


![Completed Canvas Layout](https://managedblog.github.io/managed/assets/images/22.06.09/07.CompletedCanvasLayout.png){: .align-center}

 
We need to import the Power Automate flows to our canvas app. There are a few different ways to do that, but the easiest and most functional is by adding the Power Automate pane to the canvas. This is a preview feature. Click File > Settings to open the settings menu. Select “Upcoming Features.” Scroll down the Preview menu and toggle the “Enable Power Automate pane” switch to on.

Click the Power Automate icon on the left nav bar. Click on “Add Flow.” Search for the Get User Devices flow we created in the earlier post and click on it to add it to the canvas.


![Add Get Mobile Device Flow](https://managedblog.github.io/managed/assets/images/22.06.09/08.AddGetMobileDeviceFlow.png){: .align-center}


Repeat the above step to add the RetireDevice flow to the app.


![Add Retire Device Flow](https://managedblog.github.io/managed/assets/images/22.06.09/09.AddRetireDeviceFlow.png){: .align-center}
 

All of the required elements have now been added. Select the Get Mobile Devices button, click on the dropdown menu and find the `OnSelect` property. 

When we click on this button, we want to run the Power Automate workflow to return a user’s devices. The list of devices should be returned to a collection. If the collection already exists, we want to clear it and populate it with the updated information. can clear the collection (or create a new one) using the `ClearCollect` function. This function uses the following syntax:

`ClearCollect( _[CollectionName]_ , _Values to add to collection_ )`

In this case, we will use the following formula to clear the collection `flowResults` and populate it with the devices returned from running Power Automate flow `GetMobileDevicesByUPN`:

`ClearCollect(flowResults,GetMobileDevicesByUPN.Run(txtUser.Text))`

The flow requires that we pass in the value of the UserPrincipalName. In this case, the `Run` action includes the parameter `txtUser.Text`, which will pass in the value stored in the selected user text box.


![Get Mobile Devices On Select Action](https://managedblog.github.io/managed/assets/images/22.06.09/10.GetMobileDevicesOnSelectAction.png){: .align-center}
 

The gallery should reflect the list of devices returned from our flow. We can set the value in the gallery by changing the source of the data used in the gallery. Select the devices gallery and click on the “Data Source” property. Set the data source to the newly created collection, `flowResults`.


![Set Gallery To Flow Results](https://managedblog.github.io/managed/assets/images/22.06.09/11.SetGalleryToFlowResults.png){: .align-center}
 

The Data source will now show the updated source value, but the fields need to be set to reflect the correct values in the `flowResults` collection. Click edit on the fields property.


![Updated Data Source Click Edit](https://managedblog.github.io/managed/assets/images/22.06.09/12.UpdatedDataSourceClickEdit.png){: .align-center}
 

We can now set the values of the Title and Subtitle properties of the gallery to match the values returned from the flow. Set the subtitle to show the ID returned and the Title to show the Device.


![Update Fields](https://managedblog.github.io/managed/assets/images/22.06.09/13.UpdateFields.png){: .align-center}
 

Click on the gallery name in the tree view. We want to highlight the selected item in the gallery. We can do that using an if statement. If statements use the format: 

`If( _Condition_ , _Action If True_ , _Action if False_ )`

Select the Template fill property from the list of properties, and set the value to: 

````
    If(

        ThisItem.IsSelected,
        RGBA ( 242 , 241 , 239 , 1),
        RGBA ( 0 , 0 , 0 , 0)

    )
````


![Update Template Fill](https://managedblog.github.io/managed/assets/images/22.06.09/14.UpdateTemplateFill.png){: .align-center}
 

Finally, we are ready to set the `OnSelect` property of the retire device button. In this case, we are going to run the RetireDevice flow. That flow requires the managed device ID, which is stored in the `flowResults` collection and displayed in the `ID` field of the gallery. We also want to return feedback to the app if the action was successful.

RetireDevice makes a `POST` request to retire a managed device. A successful response will return a status code of `204`, which means the action was successful, but no content was returned. We will use an `If` statement to run the flow and check the return code. If it returns a response of `204`, we will use the `notify` function to return a message to the app.

Enter the following command in the button’s `OnSelect` property:

`If((RetireDevice.Run(Gallery1.Selected.ID).result=”204”),Notify(“Device Retire Successful”)`


![Retire Device On Select Action](https://managedblog.github.io/managed/assets/images/22.06.09/15.RetireDeviceOnSelectAction.png){: .align-center}
 

The app is now complete and ready for testing. Click on the preview button on the ribbon to run the application.


![Click On Preview Button](https://managedblog.github.io/managed/assets/images/22.06.09/16.ClickOnPreviewButton.png){: .align-center}
 

Begin typing a user principal name to search for user. After you begin typing, it should automatically search for a user and begin populating the list. Click on a username to select a user. 


![Search For A User](https://managedblog.github.io/managed/assets/images/22.06.09/17.SearchForAUser.png){: .align-center}
 

The user will appear in the selected user text box. Click “Get Mobile devices” to return the selected user’s mobile devices.


![User Returned Click Get Devices](https://managedblog.github.io/managed/assets/images/22.06.09/18.UserReturnedClickGetDevices.png){: .align-center}
 

The user’s mobile devices will be returned. Select a mobile device. After selecting a device, it should be highlighted. Click “Retire device” to retire the mobile device.


![Select Device And Click Retire](https://managedblog.github.io/managed/assets/images/22.06.09/19.SelectDeviceAndClickRetire.png){: .align-center}

 
After the flow has completed successfully, a message will be returned to the app stating it has been retired successfully. 


![Device Retire Successful](https://managedblog.github.io/managed/assets/images/22.06.09/20.DeviceRetireSuccessful.png){: .align-center}
 

### Yahtzee!
----

The last three blog posts have provided a step-by-step tutorial on how to build the complete Intune Management Power App we built in our session at MMS. It was a click by click demonstration and was designed to help an endpoint administrator understand how they can use Microsoft Graph, Power Automate, and Power Apps to build their own applications for managing Intune. It is not meant to be a finished product, rather a jumping off point. I have also shared a more complete version of the app with additional features, which I talk about in [this post](https://www.modernendpoint.com/managed/Install-MMS-Demo-PowerApps-for-Intune/).

I will continue to explore Power Apps in future posts and share additional tools that I have built. Those posts won’t be designed as click by click tutorials, but I will share insights into those apps and the steps I am taking as I go. Bookmark this tutorial and come back to it if you have any questions about the basics – but I hope you will continue to explore on your own!

Keep following for more great content on building tools and automations for managing Microsoft Endpoint Manager using Graph, Power Platform, and more! 


----

This post has been the latest post in my series on automating endpoint management tasks with Microsoft Graph and the CM AdminService. I have a few more planned posts in this series, but even once the series has been completed I will continue to explore Power Platform, Microsoft Graph, and the AdminService. 

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
17. [MMS Intune Management PowerApp Demo Part 3: Adding the buttons, gallery, and completing the app](https://www.modernendpoint.com/managed/https://www.modernendpoint.com/managed/MMS-Intune-Management-PowerApp-Demo-Part-3-Adding-the-buttons-gallery-and-completing-the-app/)







