---
title: "Splatting with Invoke-RestMethod in PowerShell"
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