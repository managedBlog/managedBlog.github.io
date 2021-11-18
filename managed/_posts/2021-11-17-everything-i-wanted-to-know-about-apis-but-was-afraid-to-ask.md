---
title: "Everything I Wanted to know about APIs but was afraid to ask"
excerpt: "Learning how to use an API wasn’t hard. Finding the right resources to get started was. This post is the first in a series on how to use and access APIs, specifically Microsoft Graph and the Configuration Manager AdminService. "
header:
    overlay_image:
    teaser:
tags:
  - Microsoft Endpoint Manager
  - Intune
  - Microsoft Graph API
  - AdminService
  - PowerShell
---

_<small>Hello, and welcome back! It has been months since my last blog post, and more than a year since my last technical post. That doesn’t mean I haven’t been busy. If you had visited my blog before you may have noticed that modernEndpoint.com got a facelift and was recently moved to GitHub pages. In March I changed jobs and began working as a consultant. In October, I had the opportunity to speak at MMS Miami Beach. Speaking at MMS was not only a great experience, but it helped to provide the content for my latest blog series. I presented on automation in Microsoft Graph and the AdminService with Adam Gross. This series will be based on that presentation and is meant to provide the building blocks to build automation tasks based on those APIs.</small>_

| ![Presenting on Graph API with Adam Gross!](https://managedblog.github.io/managed/assets/images/21.11.17/000AdamG.png) |
|:--:|
| <small>Presenting on Graph API with Adam Gross at MMS Miami Beach!</small>|

>“Graph API will put the entire Microsoft ecosystem at your fingertips! You can build solutions that will make managing your resources in the cloud so much easier! There’s literally nothing you can’t do!” – Some Ignite presenter, probably
____

The early marketing for Microsoft Graph was amazing. It didn’t take me long for me to realize that it would open a lot of opportunities to automate workloads in our company. I wanted to dive in and learn more, but unfortunately, I couldn’t find documentation that was targeted to systems administrators. Most of the resources that were available were targeted at developers. It didn’t translate well to the work I was trying to do. That meant that the barrier to entry for using Microsoft Graph was high. Unless there was a business case, I wouldn’t be able to justify taking the time to learn everything I would need to know. 

Learning how to use an API wasn’t hard. Finding the right resources to get started was. This post is the first in a series on how to use and access APIs, specifically Microsoft Graph and the Configuration Manager AdminService. My goal with this series is to answer the questions I was too afraid to ask. I had a lot of questions, but I didn’t always have resources that were targeted at the work I was doing. I am sure that I was not alone in that, so I hope to provide the tools that I could never find. Starting with this post, I will lay out everything that I have learned (and am continuing to learn) about working with various APIs. These are the building blocks of automation and toolmaking with Microsoft Graph, the AdminService, and beyond. 

The basics
----

What good are building blocks without a good foundation? There will be technical deep dives in future posts, but this post is going to start at the very beginning - answering a few basic questions about Microsoft Graph.

- _**So, what is Microsoft Graph API?**_

    Microsoft Graph is a REST API built on the OData protocol.

- _**Oh, that’s cool. So… what does that mean?**_

    I did promise that I was going to keep this basic, so let’s break it down:

    An API is an Application Programming Interface. Put simply, it is a set of tools that can be used to communicate with an application. Most common APIs are text based and used to access back-end services.

    A REST API is a web service that allows a client (the requestor) to access resources on an endpoint (the server). REST stands for Representational State Transfer. RESTful services allow requestors to interact with textual resources using a predefined set of operators. REST servers are stateless, which means no client context is stored on the server. The requests themselves supply all the relevant client information in the request.

    OData stands for Open Data Protocol. It is a standard that defines the best practices for building and interacting with REST APIs. APIs that are built on OData have common structures that make it easier to explore the underlying data.

- _**That’s starting to make sense, but how does that apply to me?**_

    By understanding the basics of how REST APIs are built we can begin to put them into use in our environment. In future posts I will discuss how to explore a REST API to discover the various resources that you can access. All REST APIs use standard elements when making calls:

    - The Endpoint (URL) is the location where the web service is listening for a request.
    - The Method is a verb that describes the type of request being made. Common methods include:

        - GET – returns a resource
        - POST – creates a resource
        - PUT – updates a resource by replacing all its properties
        - PATCH – updates a resource by changing a single property
        - DELETE – removes a resource

    - Headers include information about the requestor, including client information and the authentication token.
    - The Body includes a payload with information about the request. It is not always required. When accessing Graph and the AdminService the body will usually be delivered in a JSON payload.

Breaking down a basic request
----

Now that we you understand the basics of Microsoft Graph, let’s break down a couple simple requests. In our first example we will use the Graph Explorer to return information about our user object from Azure Active Directory. There are a lot of different ways to access Microsoft Graph. For this post we will use Graph Explorer. Graph Explorer is Microsoft tool that lets you make API calls to Microsoft Graph and see the response. There are other features available that make Graph Explorer an important tool for understanding Microsoft Graph, but those will be covered more in depth in a later post.

Open the Graph Explorer by going to [https://aka.ms/ge](https://aka.ms/ge) in a browser. In the upper left corner click on “Sign in to Graph Explorer” button. If you don’t sign in you can make test calls to a sample account, but by signing in you will be able to make calls against your own tenant.

![Sign into Graph Explorer](https://managedblog.github.io/managed/assets/images/21.11.17/001.authentication.png){: .align-center}

Sign into Graph Explorer with your user principal name and password. 

![Enter your UPN](https://managedblog.github.io/managed/assets/images/21.11.17/002.signin.png){: .align-center}

If your organization has never consented to using Graph Explorer, you may be prompted to consent to allow the application to access your profile information. If you are signed in as an administrator, you can consent for your entire organization. For this example, we will click “Accept.”

![Consent to Graph Explorer](https://managedblog.github.io/managed/assets/images/21.11.17/003.consent.png){: .align-center}

Graph Explorer has been pre-populated with a basic API call. The first option is the method dropdown box. In this case we are making a GET call. The second box is the Microsoft Graph version. There are two versions of Microsoft Graph that we can access, v1.0 and beta. You can use either version, but remember the beta endpoint is subject to changes. Finally, we have an address bar where we can enter the URL for the endpoint we would like to access. 

![The initial MS Graph Query](https://managedblog.github.io/managed/assets/images/21.11.17/004.initialquery.png){: .align-center}

The default endpoint for this query is `https://graph.microsoft.com/v1.0/me`. This query returns the querying user’s Azure AD user object. In this example we returned the logged in user’s user object:

![GET me results](https://managedblog.github.io/managed/assets/images/21.11.17/005.result.png){: .align-center}

In the upper left corner, the response status is displayed. This request took 224 ms, and returned a status of `200`, which means it was successful. 

On the left side of Graph Explorer, we see several sample queries. Scroll down to users, expand it, and select “Patch me.” We will use this sample to make another API call to PATCH our user object. 

![Query Samples](https://managedblog.github.io/managed/assets/images/21.11.17/006.querysamples.png){: .align-center}

The GET request above didn’t include a body. Requests that will create or update objects will require information about the update to be made. In this case when we selected the sample query it changed the selected method to “PATCH,” and filled in the body with a JSON payload. For this example we will update the user’s department. The endpoint is still pointing at `https://graph.microsoft.com/v1.0/me` because we are updating the logged in user’s AD User object.

![Patch query](https://managedblog.github.io/managed/assets/images/21.11.17/007.patchquery.png){: .align-center}

If we click on the “Request headers” tab, we can see that this request includes the content type in the headers. This tells the service the type of payload that is being included in the request. Here we see that our content type is `application/json`.

![Patch query headers](https://managedblog.github.io/managed/assets/images/21.11.17/008.patchheaders.png){: .align-center}

Finally, we can click on the Access token to see the token that is being passed.  Each request passes an access token to the service with information about the requestor and their permissions. The token has a limited lifetime, but we can use it in other services to make requests until it expires. We can also view the contents of the token by clicking on the bracket icon {}. I will discuss the token more in depth in a later post.

![Access token](https://managedblog.github.io/managed/assets/images/21.11.17/009.token.png){: .align-center}

After clicking on the “Run query” button, we receive a response that says, “No content” with a status of `204`. This looks like it was successful, but since there was no content returned, we need to return the user object to see if the department changed. 

![PATCH me response](https://managedblog.github.io/managed/assets/images/21.11.17/010.patchresponse.png){: .align-center}

When we run the original query above no department was returned. By default, not all properties are returned. Query options can be added to the end of the URL to change the information that is returned.  Returned properties can be changed by using a select query option. Changing the selected properties can be done by appending `?$select=property1.property2,property3` to the end of the original query.

![GET query with select](https://managedblog.github.io/managed/assets/images/21.11.17/011.selectquery.png){: .align-center}

Here we can see that we received the response “OK” with a status code of `200`. The response includes the properties that we included in our select query above. We can also see that the department was successfully updated.

![GET me with select query](https://managedblog.github.io/managed/assets/images/21.11.17/012.finalresponse.png){: .align-center}

This blog post was intended to be an entry point for using Microsoft Graph. We established the foundation for future blog posts that will provide additional building blocks for workings with both Graph and the AdminService. Each post will provide another tool to help you build confidence. Later posts in this series will dive deeper into more complex tasks, including building tools to help manage your environment and automate your endpoint management workloads.

Thank you for visiting! Please keep watching for more great content!
