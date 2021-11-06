---
title: " Walkthrough: Using Guided Scenario to Deploy App Protection Policies"
excerpt: "In case you haven't explored the new Admin Center you may not have noticed the new guided scenarios."
header:
    teaser:
tags:
  - Microsoft Endpoint Manager
  - Intune
  - About Me
gallery:
  - url: https://managedblog.github.io/managed/assets/images/legacy/walkthrough/13-data-protection_orig.png
    image_path: https://managedblog.github.io/managed/assets/images/legacy/walkthrough/13-data-protection_th.png
    alt: "Data Protection Settings"
    title: "Data Protection Settings"
  - url: https://managedblog.github.io/managed/assets/images/legacy/walkthrough/14-access-reqs_orig.png
    image_path: https://managedblog.github.io/managed/assets/images/legacy/walkthrough/14-access-reqs_th.png
    alt: "Access Requirements"
    title: "Access Requirements"
  - url: https://managedblog.github.io/managed/assets/images/legacy/walkthrough/15-conditional-launch.png
    image_path: https://managedblog.github.io/managed/assets/images/legacy/walkthrough/15-conditional-th.png
    alt: "Conditional Launch"
    title: "Conditional Launch"
---

_Welcome to the managed.modernEndpoint.com blog! Posts will range from step by step walk throughs (like this one) to deep dives on topics I find interesting or am actively working on. I will also spend time discussing my perspective on Managing Modern Endpoints, a topic I am truly excited about!_

In case you haven't explored the new Admin Center you may not have noticed the new guided scenarios. These are currently still in Preview, but I wanted to do a brief walk through on one of the available scenarios. Two of the scenarios (Deploy Edge for Mobile and Deploy a Cloud Managed PC) are available on the Home blade. My personal favorite, Secure Office apps for mobile, is neatly tucked away on the Troubleshooting blade under Guided scenarios. The various guided scenarios enable administrators to quickly deploy policy sets that contain baseline policies for several device types at once.

Deploying consistent applications across device types can be a time consuming experience. This guided scenario helps both seasoned veterans and new administrators quickly deploy baseline app protection policies that cover multiple applications and device types in one workflow. We can select the applications we want to target, configure a handful of basic settings, and assign the policy to targeted groups.

​To find the new guided scenarios select Troubleshooting + support and click on "Guided scenarios." Currently there are three scenarios available. I expect that as the MEM Admin Center matures that Microsoft will make additional scenarios available. I expect that both existing and new scenarios will offer expanded functionality and cover even more complex scenarios.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/01-overview_orig.png" alt="Guided scenarios in the MEM admin center">
</p>

___

​​Each guided scenario is displayed on a card, with a brief description, link to learn more, and an estimated time to complete the scenario. To begin walking through a scenario click on start.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/05-basics.png" alt="Policy basic information tab">
</p>

___

When you start a scenario you are taken to an introduction page. The introduction offers a description of the scenario and the impact of deploying the scenario. In this case we can see that the policy will configure an App protection policy, require a PIN to launch, and require users to reset their PIN after 5 failed attempts.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/03-intro.png" alt="Secure Office Apps for Mobile Introduction">
</p>

___

One of my favorite features of the Guided Scenarios (and several other pages on the new Admin Center) is that they use breadcrumbs to allow you to jump back and forth between pages. The blades that we saw on the Azure Portal got unwieldy. When you were several pages deep in a policy you couldn't easily go back and see what you had previously configured. Through the use of these breadcrumbs you can jump to any page you have already configured, review settings, and make changes. On the surface this is a small change, but it really does make the workflow much smoother.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/04-breadcrumb_orig.png" alt="Navigation Breadcrumbs">
</p>

___

​​On the Basics tab you are prompted to add a Prefix. The Prefix will be used to name the policies that are created through the guided scenario. Each of the policies created will be added to a policy set and have a standard name. This makes finding with the policies created through a guided scenario easy.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/05-basics.png" alt="Policy basic information tab">
</p>
___

​On the Apps tab you can select the applications you want to protect. There is a baseline set of Applications pre-selected. Click Next to accept the default set of applications, or click "+ Select Public Apps" to add additional applications.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/06-apps.png" alt="Mobile Apps Baseline">
</p>

___

If you click to add additional apps a flyout appears on the right side of the screen. The functionality is the same as user and group assignment screens on policies or application assignment. Applications selected at the top will appear on the selected apps list, apps can be removed by clicking "Remove" on the list at the bottom.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/07-targeted-apps.png" alt="Selecting Additional Applications">
</p>

___

Help links open in a flyout similar to the application selection screen above rather than in a new tab. This includes the description and important notes. This is another small change that makes a big difference in the workflow

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/08-details-flyout.png" alt="Help text appears on a flyout">
</p>

___

The Configuration tab has settings that you can customize within a guided scenario. Currently there are only a handful of settings available in the Secure Office mobile apps scenario. When the app protection policies are created through the policy there is a full set of baseline settings that are set. This is one area that I would like to see built out further. I would like to see more Data Protection settings included, because those are the ones I feel I spend the most time on right now. This is a good start though, and helps you to get a feel for how these scenarios are designed.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/09-configuration.png" alt="App Protection Settings Configuration">
</p>

___

The Assignments tab is very similar to most other Assignments tabs in the Azure console with one notable exception. There is only an option to Include groups. There is no option to exclude users or groups on this screen, so if your policy will have a high impact you need to be cautious not to assign the policy to users who should not be targeted.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/10-assignments.png" alt="Assigning App Protection Policies">
</p>

___

The Review + create tab lets you confirm the settings you have configured. This is where the breadcrumbs at the top of the screen really shine. If you see settings that need to be updated before creating the policy you can quickly navigate to the correct page, update the setting, and return here to create the policies.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/11-review.png" alt="Review and create tab">
</p>

___

The confirmation page gives you details on what items were created. This guided scenario creates a policy set that includes the application protection policies that we just set up. I will take a deeper dive into the policy sets in a future post - but here we can see one that was automatically created and some of the policies that can be included.

<p align="center">
    <img src="https://managedblog.github.io/managed/assets/images/legacy/walkthrough/12-confirmation.png" alt="Confirming configured policies">
</p>

___

This guided scenario created both an iOS and an Android App Protection policy. When we open the Android Policy we can see the settings we selected and the other baseline configurations that were included. It has all of the typical policy groups - Data Protection, Access Requirements, and Conditional Launch. Once the policies have been created you can edit them like any other policy. 

{% include gallery %}

___

There has been a lot of discussion around the new Modern Endpoint Management Admin Center. These guided scenarios are one of the unheralded changes that I have noticed. They are still in preview and are focused on more basic scenarios. I believe that these guided scenarios could really have a huge impact on how we create policies in the future. There's a lot of room for growth here, but the ability to create one baseline policy and have it apply across device types will make a time consuming workflow much faster with more consistent results.