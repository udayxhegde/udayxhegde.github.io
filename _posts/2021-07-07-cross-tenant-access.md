---
layout: post
current: post
navigation: True
title: Cross tenant resource access
date: 2021-07-07 07:00:00
class: post-template
subclass: 'post'
author: uday
---

## Introduction

A multi-tenant application in Azure Active Directory provides a convenient way for developers to serve multiple customers with a single instance of their application. For the application to do anything interesting, it needs data unique to each customer. The multi-tenant application model allows the single instance of the application to access data in multiple other tenants.

A typical multi-tenant application is configured to access tenant-specific data in the context of the logged-in user. This is called delegated access since the application accesses the data as the signed-in user. The application doesn't even need to know which tenant it is accessing, since the signed-in user who is interacting with the application belongs to a tenant and that implies which tenant is being accessed on behalf of the user. 

Rarely, some people have scenarios where they need their application to run without a signed-in user, and still access resources in multiple tenants. In addition, there are some scenarios which deal with Azure resoures, which need to be treated differently when compared to Microsoft Graph.

Since the multi-tenant application model is the only way you can access cross-tenant resources from a single instance of an application, it is handy to be able to use it for these not so common scenarios.

This blog post explores how we can try to achieve this.

## How the multi-tenant app works
To ground ourselves, let's briefly look at how a multi-tenant app works for the common scenario where there is user interaction.

First, you register the application in the developer's tenant and configure what permissions are needed by the application. In most cases, this is data in Microsoft Graph, and some example permissions are User.Read.All or Group.Read.All.

Then, the developer codes the application to access the data. The developer is not required to know which tenant to access ahead of time, that can be figured out dynamically based on the user's tenant when a user is interacting with the application. So the developer simply makes an authentication request to the common endpoint in Azure AD ((https://login.microsoftonline.com/common), and Azure AD will provide the appropriate context for the data access based on the signed-in user.

Next, an administrator in another tenant can add this application to their tenant and consent to the application being granted access to the data it needs in that tenant. 

Finally, when a user in that tenant accesses the application, the application accesses the data it needs and provided it was granted consent for that data, it is able to access it in the context of the user.

## Challenges we need to overcome in the app only flows

Now that we briefly looked at how a typical multi-tenant app works, let's consider the challenges when we don't have a user interacting with the application.

1. We now need to tell the app ahead of time which tenants it needs to access. Unfortunately, there is no capability right now to detect in which tenants the application has been added.

2. Unlike Microsoft Graph, which is a single instance per tenant, Azure resources are different: there are multiple instances of a single resource in a tenant. So you also need to identify which specific instance of an Azure resource the application should access.

3. The user's authentication when accessing the application served the purpose of authenticating both the user and the application. In the absence of the user, we need to add credentials to the application for it to authenticate as itself.

## Building a multi-tenant app with the app-only flow, accessing Graph or Azure resources


### Step 1: Create an application, with credentials.

Since the application is running without user interaction, we need to allow it to authenticate by itself. This requires us to configure it with credentials it can use to authenticate. 

1. Create your multi-tenant app in the developer tenant. Go to the Azure AD portal, click "App registrations", and click on "+ New registration".  Give a name for your application, and then pick "Accounts in any organizational directory (Any Azure AD directory - Multitenant)" option. Since this is an app without any user interaction, we don't need to configure the redirect URI.

2. Clicking register creates the application registration and puts you on a page where you can configure the application. Make a note of the Application (client) ID on this page, you will need it later. Go to Certificates & secrets, and add a new client secret. Save this securely in a key vault in the developer's tenant.


### Step 2: Configure Microsoft Graph permissions needed by the application
This step is only needed if the application needs to access Microsoft Graph in another tenant. If there is no such need, skip this step

1. On the same page where you landed after registering the application, click on "API permissions". Pick "Application permissions" in "What type of permissions does your application require". And then Pick the permissions you want to add (eg: user.read.all).

![Adding permissions to an application](/images/apppermission.png)


### Step 3: Add the created application in other tenants where it is needed. 

This has to be done by an administrator of each of the tenants. While there are several ways to achieve this, Azure CLI provides a simple way to do this, especially since we are not configuring the application with user interaction.

1. az login --tenant {tenantid} --allow-no-subscriptions

2. az ad sp create --id {your application id from Step 1}

### Step 4. Consent to the application, to grant it permissions to Microsoft Graph

This is only if you have configured the application with permissions needed in Step 2, otherwise, skip this step.

1. Go to the Azure AD portal in the target tenant, and go to Enterprise Applications.

2. Pick Application Type "All Applications", enter the application id of the application in the search bar, and click Apply to find the application that was added in Step 3.

3. Click on the application, and click on "Permissions". Click on "Grant admin consent for {tenant}". Make sure you sign in as an admin with permission to grant consent, and click Accept in the consent screen

![Consent screen](/images/consent.png)


### Step 5: Accessing Azure resources. 

1. Go to the Azure portal in the target tenant and pick the azure resource to which you want the application to be granted access. Grant your application access to the Azure resource. How you do this may vary a bit depending on the application and even the permission type. For example, for KeyVault, go to settings, Access Policies, and Add an Access policy

![Adding an access policy to keyvault](/images/keyvaultpolicy.png)


### Step 6: Code and deploy your application
At this point, you have completed all the necessary configurations needed for the application to access the resources across tenants. 

The main special sauce you need in the application code is the following:

1. List of tenants and instances of Azure resources the application should access. This is needed since we can't figure this out dynamically (at least not yet in Azure AD).

2. Authenticate the application at the tenant endpoint (https://login.microsoftonline.com/{tenantId}) using the OAuth Client Credentials grant flow. Both MSAL and Azure/identity SDK have support for this flow. 

To see a very rudimentary example that illustrates the code in Node, see [this github repo](https://github.com/udayxhegde/multitenant-daemonapp-node). It shows how you can achieve this for both Microsoft Graph and Azure KeyVault.


