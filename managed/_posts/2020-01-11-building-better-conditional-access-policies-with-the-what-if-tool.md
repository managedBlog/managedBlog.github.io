---
title: "Building Better Conditional Access Policies with the What If tool"
excerpt: "Have you ever deployed a Conditional Access policy, only to later discover that users had found a way to circumvent it? It is surprising to discover that someone found a way around your carefully designed and tested policy."
header:
    overlay_image:
    teaser:
tags:
  - Microsoft Endpoint Manager Admin Center
  - Intune
  - Conditional Access
---

Have you ever deployed a Conditional Access policy, only to later discover that users had found a way to circumvent it? It is surprising to discover that someone found a way around your carefully designed and tested policy.

Setting up Conditional Access policies can get confusing. If you don’t plan your policies carefully, you may end up with holes that you didn’t expect. Most of us start by thinking about the scenarios we want to allow. The problem with this approach is that if you only think about what you want to allow you may forget to block scenarios you don’t approve. Without a policy in place to block unapproved scenarios, a malicious actor can exploit those scenarios by simply providing a user’s compromised credentials.

For example, consider this scenario: You are asked to create a policy that will allow a user to access their email on an iPhone. You will allow a user to use the native mail application with ActiveSync or Outlook if the application is protected by a Mobile Application Management policy. The policy you create is applied to the user, targeted to Exchange Online, and applies to ActiveSync and Modern Authentication clients. Controls are set to grant access if the device is compliant or if a MAM policy is applied.
​
When you test the policy and the device is not registered you are blocked from accessing the application in the ActiveSync client. The Outlook application prompts you to install the Azure Authenticator to access the application. Once the Authenticator app is installed or it is enrolled the user can access their mail in the targeted applications. This policy works as designed; users must meet one of the controls in the policy to use their email as intended. You may consider the policy successful and move it into production. However, there’s just one problem…
When a conditional access policy is in place all conditions must be met before the access controls are evaluated. If you had tested other scenarios that you expect should fail, you would have found that in this case the user could access their email from the device’s browser – even if it is not enrolled in Intune. The policy would have been evaluated as follows:​

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/01-table-unblock-sign-in.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">Intune Conditional Access policies won't evaluate scenarios that aren't accounted for.</figcaption>
</figure>

​The controls to grant access are not evaluated because the policy doesn’t apply to browser access on iOS devices. Sign-in is allowed. If this user’s credentials are compromised and no other controls are in place, a malicious actor would be able to access the mailbox. Since the browser is also not being managed through Intune this represents a significant data leakage risk.

Any scenario that has a policy applied to it will block access if the conditions are not met. Scenarios that do not have any policy applied will allow access to anyone with a valid credential. We can prevent users from accessing services by using a second policy with the block access control. In this instance we could successfully block access from unapproved scenarios with a policy that applies to Exchange Online on iOS devices. We would include a condition that would target the browser and other client applications. The control would be set to block. That policy would be evaluated as follows:

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/02-blocked-sign-in_orig.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">By adding a policy block unapproved scenarios we can use Conditional Access to block scenarios not covered in other policies.</figcaption>
</figure>

A proper Conditional Access deployment needs to consider scenarios that should always be blocked. As policies get more complex, it can become more difficult to determine if you have accounted for all possible scenarios. Intune and the new Modern Endpoint Manager Admin Center have a built in What If tool that lets you walk through a range of possible scenarios to see how your policies will be evaluated.

To demonstrate this tool, I will use a more complex example. In this case we want to create a policy that will apply to both iOS and Android devices. iOS devices can use Exchange ActiveSync if they are Intune Compliant. Users should be able to access their mail through the browser on compliant mobile devices. We want to allow Outlook mobile on any iOS or Android device whether it is enrolled or not. There is a Mobile Application Management policy in place. ActiveSync should be blocked on Android devices. We also want to block access from other email clients in all scenarios.

We will use 4 policies to accomplish this goal:

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/03-scenarios_orig.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">Stacking Conditional Access policies in Intune let us cover a wide range of scenarios.</figcaption>
</figure>

In a future blog post I may create a walkthrough on how I created these policies. You can find more information on [configuring Conditional Access here](https://docs.microsoft.com/en-us/intune/protect/conditional-access-exchange-create). 

You can access the What If tool on the Conditional Access blade by clicking on the button on the page ribbon.

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/04-what-if_orig.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">Click the What If button on the Microsoft Endpoint Manager ribbon on the Conditional Access page to access the tool.</figcaption>
</figure>

The What If tool has fields that allow you to build a scenario to test. You can select a user, application, IP Address, Country, device platform and state, client app, and Sign In risk. 

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/05-what-if-overview_orig.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">The What If tool lets you test various scenarios to see which of your policies will apply to your managed devices.</figcaption>
</figure>

I want to test whether I will be able to access Exchange Online from my enrolled iPhone through the Outlook application. To test this scenario, select the appropriate settings and click the blue What If button.

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/06-setting-up-what-if-scenario.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">Configure the Conditional Access scenario you want to test.</figcaption>
</figure>

When you click on the button the evaluation results of your scenario will appear below. There are two different tabs – Policies that will apply and Policies that will not apply. The policies that will apply shows any policies that will apply and the controls that will be applied to your device. In this case the Enrolled Mobile Device Policy will apply with access being granted because the device is compliant.

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/07-policies-that-will-apply_orig.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">View Policies that will apply in the What If tool in the Microsoft Endpoint Manager admin center.</figcaption>
</figure>

The policies that will not apply tab shows all the policies in your tenant that will not apply. It also shows the reason the policy will not apply. This is helpful because you may have a policy you expected to apply but did not. You can easily edit a policy within the tool and re-evaluate it. 

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/08-policies-that-will-not-apply_orig.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">The Policies that will not apply tab helps to identify why Conditional Access policies didn't apply as intended.</figcaption>
</figure>

If we were to change the above scenario to test on an Android device, we expect that access will be blocked. We can see that two different policies were applied: Enrolled Mobile Device Policy and Block EAS on Android. The first policy would grant access, but the second policy explicitly blocks access. When using Conditional Access, a block condition will always take precedence over an allow condition.

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/09-multiple-policies-on-android_orig.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">Multiple Microsoft Endpoint Manager Conditional Access policies can apply to the same device. Block conditions take precedence.</figcaption>
</figure>

If we wanted to edit a policy because of unexpected or undesirable results, we can click on the policy here to edit it. Clicking on a policy will open an editor flyout on the right side of the page. Here you can make changes and save a policy without having to leave the What If tool.

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/10-flyout-editor.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">Edit Conditional Access policies on the fly within the What If tool.</figcaption>
</figure>

After you save changes the policy will be re-evaluated. Any changes made will be reflected on the Evaluation results.

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/11-policy-after-changes-reevaluated_orig.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">After editing Conditional Access policies with the What If tool the results are immediately re-evaluated.</figcaption>
</figure>

To effectively use the What If tool, make sure to test all the possible scenarios for the device and application you are targeting. This will ensure that all scenarios are accounted for. It is much quicker than doing it manually and waiting for a device to download updated policies. This will streamline your testing process and help you to build better policies.

{% capture fig_img %}
![Special thanks to @rundavidrun]({{ "https://managedBlog.github.io/managed/assets/images/legacy/ConditionalAccess/12-block-unapproved-scenario.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption style="font-style: oblique; font-size:-4; text-align:center">Make sure to test blocked scenarios in the What If tool to ensure Conditional Access policies block unwanted traffic.</figcaption>
</figure>

Properly built Conditional Access policies are one of the most important tools to protect corporate resources accessed through Azure and Office 365. Taking the time to plan your policies and thoroughly test them is essential. The What If tool gives you the peace of mind to know that you have built the best possible policy set for your environment.
