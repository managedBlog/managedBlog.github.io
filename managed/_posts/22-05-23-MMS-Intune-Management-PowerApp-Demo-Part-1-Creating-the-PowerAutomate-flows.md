---
title: "MMS Intune Management PowerApp Demo Part 1 Creating the PowerAutomate flows"
excerpt: "In this blog post, I will explain why the demonstration in my last post wasn’t practical for automation and provide a PowerShell runbook that solves those problems."
header:
    og_image:
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


Hello and welcome back! Now that MMS is over, I feel like I can finally focus on writing blog posts again! This post is the latest installment in my series on automating endpoint management tasks with Microsoft Graph and the Configuration Manager AdminService. If you attended MMS, you may have joined some of the sessions that covered PowerApps, Microsoft Graph, or the AdminService. If you did, the next few posts make look (and sound) familiar.

Last week, [I shared a post](https://www.modernendpoint.com/managed/Install-MMS-Demo-PowerApps-for-Intune/) that included instructions on how to import a Power App that I used in one of my demos. That app provides a basic GUI interface for an administrator to find a user’s devices, select one, and retire it from Endpoint Manager. It demonstrates the art of the possible for administrators that are interested in exploring PowerApps.  

Over the course of the next few posts, I will walk you through how that app was created, including the controls that were added to the Power App and the underlying Power Automate flows. The goal of these posts is to share a basic how-to for creating your own apps built on the Power Platform. It will also lay the groundwork for future posts in this series. 


### I’ve got the power!
----

If you aren’t familiar with the Power Platform, it consists of four services that work together to combine multiple technologies into a single platform to help streamline modernization tasks:

1.	Power BI is used to help visualize business intelligence data for building reports and dashboards.
2.	Power Automate is a low-code tool that lets users from across your organization create automation workflows that can connect various systems in your environment.
3.	Power Apps can be used to create custom applications to streamline business processes. It’s low-code format allows less technical users to create tools that are custom-built for your environment.
4.	Power Virtual Agents can be used to create intelligent chatbots that are built on your business processes.

All these tools are powerful (as their names suggest), and each of them have their place in process automation. With that being said, I have no plans to cover Power BI in the future, and while I hope to eventually cover Power Virtual Agents, that won’t likely happen as part of this series. 

However, that leaves both Power Apps and Power Automate. Both tools will be covered in depth over the next few blog posts. [I have covered Power Automate in a previous post](https://www.modernendpoint.com/managed/Using-Power-Automate-to-update-MEMCM-Device-based-on-Intune-User/), and will continue to cover it throughout this series. I have several posts focused on Power Apps planned, and those posts will be accompanied by tools that will be shared with the community. Whenever I build a tool to share, I will also include blog posts that explore the tool in detail so that you can see how it was built and customize it for your environment. 

Perhaps the greatest value in Power Platform is the ability to use connectors to connect to outside services. This will allow us to expand our low codes tools to other services in Azure, other cloud services, and even to on-premise resources. I will do my best to cover as many different tools as practical, but the possibilities are endless.
Today we are going to explore the underlying Power Automate workflows that are being used in the demo Power App. This app is relatively simple. It uses two connectors (O365 Users and Azure Key Vault) and two Power Automate workflows. The app searches for a user from Office 365, returns their devices, and then lets an administrator select one from a gallery object and retire it from Intune. 


### Cool, so what do we do next?
----

There’s a chicken-or-egg question when building Power Apps based on Power Automate workflows. Should I create the workflow first or the app first? I have done both. Typically, I prefer to create the base flow, and then update it based on what I learn from building the app. You will likely have to make some changes to the flow, but the Power App is more dependent on settings in the flow than the flow is dependent on the App. For the sake of this example, we are going to start by creating our Power Automate flows first.

Sign in to [Power Automate](https://powerautomate.microsoft.com/) and click + Create.

 
![Create New Flow](https://managedblog.github.io/managed/assets/images/22.05.23/01.CreateNewFlow.png){: .align-center} 


Select Instant Cloud Flow from the “Start from blank” section of the page.

 
![Instant Cloud Flow](https://managedblog.github.io/managed/assets/images/22.05.23/02.InstantCloudFlow.png){: .align-center} 


Name your flow, select PowerApps, and click create. This will create a new flow with a Power App trigger. 

 
![Power Apps Trigger Create Flow](https://managedblog.github.io/managed/assets/images/22.05.23/03.PowerAppsTriggerCreateFlow.png){: .align-center}


The app will initially be created with a single step, a PowerApps trigger. This trigger tells the app that it will be launched by an app created in Power Apps. Click New Step to create a new step.


![Power Apps Trigger Created](https://managedblog.github.io/managed/assets/images/22.05.23/04.PowerAppsTriggerCreated.png){: .align-center}


The “Choose an operation” dialogue will open. In this flow, we are going to start our flow by initializing two variables – `UserPrincipalName` and `HTTPFilter`. Our demo flow will specifically target non-Windows devices. We will use a filter to accomplish that. 

Search for “initialize” in the operation search box. Click on “initialize variable.”


![Choose Init Variable Operation](https://managedblog.github.io/managed/assets/images/22.05.23/05.ChooseInitVariableOperation.png){: .align-center}


An initialize variable step will be added to the flow. Click on the ellipses `…` in the upper right corner and select rename. I renamed my step to “Init UserPrincipalName.” This will make it easier to find later. Set the name value to `UserPrincipalName`. The type should be set to `String`. Click on the value field and select “Ask in PowerApps” from the dynamic content list.
 

![Init User Principal Name](https://managedblog.github.io/managed/assets/images/22.05.23/06.InitUserPrincipalName.png){: .align-center}


This will set the value field to reflect `InitUserPrincipalName_Value` from the PowerApps step.


![Select Init User Principal Name](https://managedblog.github.io/managed/assets/images/22.05.23/07.SelectInitUserPrincipalName.png){: .align-center}


Click New Step and add another initialize variable step. Rename the step “Init HTTPFilter.” Set the name value to `HTTPFilter`, the type to `String`, and the value to `operatingSystem ne ‘Windows’`. This step will limit the search results to Android and iOS devices. This isn’t a hard requirement but will help to limit the scope of the demo application.
 

![Init Http Filter](https://managedblog.github.io/managed/assets/images/22.05.23/08.InitHttpFilter.png){: .align-center}


This app relies on secrets being stored in Azure Key Vault. We will use an Azure Key Vault connector to get the ClientID of an Azure application registration and an associated ClientSecret. [I covered the specific secrets required in my last blog post.](https://www.modernendpoint.com/managed/Install-MMS-Demo-PowerApps-for-Intune/) Once you have created a Key Vault and added the required secrets, click new step to add a new step. Search for Azure Key Vault and select “Get secret.”
 

![Select Get Secret Step](https://managedblog.github.io/managed/assets/images/22.05.23/09.SelectGetSecretStep.png){: .align-center}


Enter your vault name and click “Sign in.” When prompted, sign in to create the connector.


![Create A K V Connection](https://managedblog.github.io/managed/assets/images/22.05.23/10.CreateAKVConnection.png){: .align-center}


Rename the step to “Get ClientID” and select your Client ID secret from the list of secrets that are available. 
 

![Get Client I D](https://managedblog.github.io/managed/assets/images/22.05.23/11.GetClientID.png){: .align-center}


Add a second get secret step, rename it to “Get ClientSecret.” Select your client secret from the list of available secrets.

 
![Get Client Secret](https://managedblog.github.io/managed/assets/images/22.05.23/12.GetClientSecret.png){: .align-center}


When we are working with secrets in Power Automate, we want to make sure that they are protected. We can limit the visibility of the secrets in the flow output by clicking on the ellipses and selecting “Settings.” On the settings screen, set “Secure Inputs” and “Secure Outputs” to on.


![Secure Client Secret Steps](https://managedblog.github.io/managed/assets/images/22.05.23/13.SecureClientSecretSteps.png){: .align-center}


Next, we will add an HTTP step. Add a new step and select, “HTTP”.


![Select H T T P Step](https://managedblog.github.io/managed/assets/images/22.05.23/14.SelectHTTPStep.png){: .align-center}


Rename the step to “GET Devices by UPN.” Set the Method to `GET`, and the URI to `https://graph.microsoft.com/v1.0/users/[USERPRINCIPALNAME]/managedDevices?$Select=deviceName,operatingSystem,id&$Filter=[HTTPFilter]`. Replace `[UserPrincipalName]` and `[HTTPFilter]` with the variables we set by selecting them from the Dynamic Content selection box.


![Get Devices By U P N](https://managedblog.github.io/managed/assets/images/22.05.23/15.GetDevicesByUPN.png){: .align-center}


Click on “Show Advanced Options.” Set Authentication to Active Directory OAuth. Set the authority to `https://login.windows.net`, the tenant to your tenant ID or primary domain, and the audience to `https://graph.microsoft.com`. For the Client ID select the `value` from Get ClientID under dynamic content. Repeat this for the client secret value. 

 
![Set H T T P Authentication](https://managedblog.github.io/managed/assets/images/22.05.23/16.SetHTTPAuthentication.png){: .align-center}


The HTTP step is the heart of this flow. We are making an HTTP request to Microsoft Graph to return a user’s managed devices. Once we make the request, we need to process the results and return the results to our flow. The next step will be to parse the JSON object that was returned. To do that, we need a sample of the JSON response from the HTTP request. To get that sample we will test our workflow. 

Click “Test” on the ribbon.


![Click Test Button](https://managedblog.github.io/managed/assets/images/22.05.23/17.ClickTestButton.png){: .align-center}


When prompted, select “manually” to test the flow manually and click the test button at the bottom of the screen.

 
![Test Flow Manually](https://managedblog.github.io/managed/assets/images/22.05.23/18.TestFlowManually.png){: .align-center}


If prompted, confirm the Azure Key Vault connection is working and then click continue.


![Confirm Azure Key Vault](https://managedblog.github.io/managed/assets/images/22.05.23/19.ConfirmAzureKeyVault.png){: .align-center}


Enter the User Principal Name of a user whose devices you want to return and then click Run flow at the bottom of the screen.


![Enter User Principal Name](https://managedblog.github.io/managed/assets/images/22.05.23/20.EnterUserPrincipalName.png){: .align-center}
 
![Click Run Flow](https://managedblog.github.io/managed/assets/images/22.05.23/21.ClickRunFlow.png){: .align-center}


Once the flow run has been completed, we can open the HTTP step. We should see a status code of 200, which tells us the request was successful and content was returned.

 
![H T T P Flow Output](https://managedblog.github.io/managed/assets/images/22.05.23/22.HTTPFlowOutput.png){: .align-center}


Copy the entire contents of the output body and click “Edit” to edit the flow. Add a new Parse JSON step. Set the content field to use the Body of the HTTP request, and then click “Generate from Sample.”

 
![Create Parse Json](https://managedblog.github.io/managed/assets/images/22.05.23/23.CreateParseJson.png){: .align-center}


Paste the output body from the HTTP request in the sample JSON dialogue and click done. (Note: In this instance, we can select a single object from the `value` in the JSON payload to use later in our flow. Copy the value, including the square brackets, to use later.)


![Insert Sample Json](https://managedblog.github.io/managed/assets/images/22.05.23/24.InsertSampleJson.png){: .align-center}


Add a new step, search for select, and select the Data Operation select step.


![Select Data Operation Select Step](https://managedblog.github.io/managed/assets/images/22.05.23/25.SelectDataOperationSelectStep.png){: .align-center}


The select step will create a new JSON object based on the output from the Parse JSON step. Set “From” to use the resulting `value` from the Parse JSON step. Add three fields under “Map”. Each value will be based on the corresponding result from the Parse JSON step. Set Device to `deviceName`, OS to `operatingSystem`, and ID to `id`.

We are using the Select operation because it will handle a JSON array. By using the `value` field from the Parse JSON step, we are telling it to parse each individual value in the array returned from the JSON output. 


![Prepare Select Step](https://managedblog.github.io/managed/assets/images/22.05.23/26.PrepareSelectStep.png){: .align-center}


Add a new step, search for response, and select the Request Response step.


![Select Request Response Step](https://managedblog.github.io/managed/assets/images/22.05.23/27.SelectRequestResponseStep.png){: .align-center}


If we were returning a single object to our PowerApp we could use the “Respond to PowerApps” step, unfortunately that option doesn’t support arrays. If we need to return multiple objects, we can do that with the Response step. The select step we used above will create a JSON array that we can pass into our response. If the flow ran successfully and it contains a response, we will want to send a status code of `200`, which tells the app that the response was successful and contains content.

Set the Body to the `Output` of the select step above. We need to include a JSON schema in the response step. In this case, the schema matches the value from the Parse JSON step above. If you copied that value you can use it again here, otherwise test your flow and copy the output value from our select step above. Click Generate from Sample when you are ready.


![Response Step](https://managedblog.github.io/managed/assets/images/22.05.23/28.ResponseStep.png){: .align-center}


Paste the sample JSON payload into the editor and click done. 
 

![Response Json Payload](https://managedblog.github.io/managed/assets/images/22.05.23/29.ResponseJsonPayload.png){: .align-center}


After saving the sample payload the flow is complete. Click Save to save the flow. 
 

![Completed Response Step](https://managedblog.github.io/managed/assets/images/22.05.23/30.CompletedResponseStep.png){: .align-center}


You can also test the complete flow at this point. If you test a workflow that ends with any sort of “response” step the final step will be skipped. Since the flow is being run manually, there is not place to send the response.


![Complete Flow Test](https://managedblog.github.io/managed/assets/images/22.05.23/31.CompleteFlowTest.png){: .align-center}


### But wait, there’s more!
----


Our Power App has three key steps – user lookup, device lookup, and performing the retire action. User lookup will be handled with a connector in the PowerApp. The flow we just built handles device lookup, but we still need to handle the retire action. We will trigger the action using a second Power Automate flow.

Triggering a device action may seem like a more complex task, but it only requires a single Microsoft Graph `POST` request. A successful `POST` request to perform a device action returns a status code of `204`, which means no content is returned. Since no content is returned, we don’t need to parse the response and the only response we need to return to the app is the status code.

This leaves us with a much shorter flow.


![Retire Device Flow](https://managedblog.github.io/managed/assets/images/22.05.23/32.RetireDeviceFlow.png){: .align-center}


Create a new flow with a PowerApps trigger. Add an initialize variable step. Rename the step. Set the name to `DeviceID`, type to `String`, and set the value to “Ask in PowerApps.”

 
![Init Device I D](https://managedblog.github.io/managed/assets/images/22.05.23/33.InitDeviceID.png){: .align-center}


Next, add in steps to get your ClientID and ClientSecret from Azure Key Vault. These will be identical to the steps used in the flow above. _(Note: In a production environment, we could optimize this so we don’t have to keep making calls to Azure Key Vault – but in the interest of keeping the demo environment simple, we aren’t doing that here.)_


![Azure Key Vault Secrets](https://managedblog.github.io/managed/assets/images/22.05.23/34.AzureKeyVaultSecrets.png){: .align-center}


We now have all of the information we need to make our `POST` request to Microsoft Graph. Create a new HTTP request. Set the method to `POST` and the URI to `https://graph.microsoft.com/v1.0/deviceManagement/managedDevices/[DeviceID]/retire`. (Make sure to replace `[DeviceID]` with the DeviceID variable we created above. Click show advanced options and configure the authentication settings to match our previous flow.


![Post Retire Action](https://managedblog.github.io/managed/assets/images/22.05.23/35.PostRetireAction.png){: .align-center}


Finally, in our app we want to give the user a notification that lets them know if the flow ran successfully or not. Since we are only returning a single object (the status code) we can use the built in PowerApp response. Create a new step and search for “Respond,” select “Respond to a PowerApp or flow.”

Set the name of the value to “Result,” and set the value to the `Status code` returned from the HTTP step. Save your flow.
 

![Respond To Power App](https://managedblog.github.io/managed/assets/images/22.05.23/36.RespondToPowerApp.png){: .align-center}


### That’s all for now!
----


In today’s post we created the Power Automate workflows that are used in the demo app that we created at MMS. Next, I will explore the Power App itself. There are a lot of different elements that go into building the app, and I want to take the time to explain them in detail. The demo app is relatively simple, but there are a lot of lessons that can be learned from it. 

We will explore how the demo app was built over the next two posts. Keep watching for those posts soon!

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
15. [MMS Intune Management PowerApp Demo Part 1 Creating the PowerAutomate flows](https://www.modernendpoint.com/managed/MMS-Intune-Management-PowerApp-Demo-Part-1-Creating-the-PowerAutomate-flows)



