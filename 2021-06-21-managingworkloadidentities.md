---
layout: post
current: post
navigation: True
title: Managing Workload Identities
date: 2020-07-01 10:00:00
class: post-template
subclass: 'post'
visible: 1
author: uday
---

<main>

# Quick read
Software workloads continue to move to the cloud as more businesses shift to cloud computing. Most of new enterprise software developed these days are built natively in the cloud. The software workloads, made up applications, scripts and services, typically need an identity in order to authenticate and access resources or communicate with other services. This has resulted in a dramatic increase in “workload identities” in cloud identity systems. 

Most cloud identity systems have a good level of maturity when it comes to securing, managing and governing user identities. However, this level of maturity currently does not exist for these workload identities. Their growing number along with the increasing awareness of damage caused due to security breaches is causing many enterprises to pay closer attention to how to manage them in a better way. They are seeking capabilities such as lifecycle management, access management, secure adaptive access and secrets management. These are all areas we are putting a lot of emphasis on as we build out these capabilities in Microsoft’s identity system. 

# And now, if you are interested in more details, read on…

## Taxonomy, and why call them workload identities.

First, let’s start with the terminology. There is no consistent terminology in the industry today for these “software identities”. In Azure AD, depending on what they are used for, they are called application identities, managed identities or service principals. In Windows Active Directory, we have service accounts which is another term that shows up in many other platforms. The plethora of names lead to a confusing outcome when people use the different terminologies to mean the same thing. To address this problem, we will call the identities required for software workloads, meaning applications, services, scripts, daemons, etc as “workload identities”. It maybe useful to frame this terminology by looking at the taxonomy of all the identities we see in Azure Active Directory.
<image>

### Human identities 
These are the ones we are most familiar with. We use them everyday ourselves, and companies use these identities to provision their employees and FrontLine Workers (FLW) with access to things they need to do their jobs. These identities also allow external users, such as business partners or end consumers, to be provisioned with the right level of access.

### non-human identities or “Machine identity”.

Machine identity is a term that is getting used by some industry analysts as well as product vendors to cover all the scenarios for non-human identities. We can classify them in four broad categories, but over time its likely that these categories may overlap or even converge. 
#### Mobile or Desktop devices
 Devices that we carry with us, either mobile or desktop, need an identity. In Azure AD, we use this identity to enable scenarios such as conditional access requiring a compliant device.  

#### Internet of things (IoT)
These devices typically act on their own, unlike a handheld device or a desktop.

#### Workload identities
There are the identities we are focusing on in this blog. They represent identities needed by applications, services and scripts to authenticate and access resources. 

#### Robotic process automation (RPA) 
RPA is a relatively new trend where human based processes are being automated. Some of these scenarios end up needing a user identity, primarily to deal with the fact that most software licensing require user identities. And in some other scenarios, they may use one of the types of machine identities.


## So, what are some problems we face with workload identities?

As the number of workload identities increase, identity administrators are starting to wonder how to manage their growing number. The regular spate of security breaches is causing an heightened awareness of the need to secure and govern these identities. Developers checking in the credentials/secrets for these identities in their code into their GitHub repos has been a common weakness that hackers are exploiting.  The most infamous of these exploits was the Uber one several years ago. This is causing IT administrators to ask all the right questions: how do we avoid compromised workload identities trying to access our systems? Why cant we use conditional access policies like we do for users? What do we need to do to cleanup these identities when they are no longer in use?
The nature of workload identities do not lend themselves to be managed in all the same ways we manage user identities. Let’s look at some of the unique differences to see why we need specialized capabilities for these identities. 

### No Joiner/mover/leaver process.
User identities have a clear process, typically called joiner/mover/leaver that is rooted in the HR system. When a user joins the company, they are added to the HR system that results in them being provisioned in the identity system. Using the information in the HR system, the user is also decorated with attributes such as their team and their role. This allows the user to be provisioned the necessary access needed for their job, for example by building a dynamic group based on these attributes. When the user moves to a different team or role, they are automatically removed from the original group and placed in a different group, gaining access to new resources while losing access to the old ones. 
Workload identities don’t have such a formal process we can rely on. They get created by someone who deploys the application or service and are generally forgotten after that. It is generally difficult to figure out if a workload identity is still in use, and what it has access to. The person who created this identity may have forgotten why they created this, may have changed teams or even left the company. So then how do you manage the lifecycle of these identities? In most cases, this ambiguity leads them to remain in the system, forever untouched.

### Need to be provisioned with credentials or secrets.
Unlike users who can typically show gestures or biometrics to prove who they are, workload identities need to be provided access to their credential so they can use it to authenticate and then get access to resources. So careful attention needs to be paid to manage these secrets. Many organizations have not developed strong guidance in this area, and it is up to individual developer or devops team to figure this out. The rotation of secrets tends to be especially challenging since that part is hard for most folks to automate, and it tends to be done rarely and manually. Given this challenge, customers need better tools to ensure these credentials are stored securely and rotated regularly. And to deal with the possibility of security breach, we also need capabilities such as anomaly detection in identity protection and conditional access policies.

### Multiple identities with fine grained access to a variety of resources
 A single workload may be configured with multiple workload identities. For example, the application identity provides it access to parts of Microsoft Graph. And individual services within the application have their own identities (aks service accounts), one of which may be provisioned to access Azure KeyVault and the other with access to Azure Storage. Given this characteristic, customers need ways to check what does an identity have access to, whether that access is needed over time and when can they be removed. 

 ## How do we help customers manage these identities in a better way?

We hear from customers and industry analysts that this is a space that is fraught with problems primarily because there are no good solutions to tackle these issues. While niche tools do exists for things such as secret management and privileged access management, they solve a very small subset of the problem space. It is quite clear to us that customers will benefit from better ways to govern, secure and manage these identities. Here are our thoughts on what we ought to do to improve this space.

### Lifecycle management
The most important part of managing the lifecycle of these identities is to capture the meta-data on who created it and why it was created, and use automation to keep that information up to date, even as employees move teams or leave the organization. In addition, we complement this information with all the activity from that identity, such as when it last authenticated and requested a token. 
### Access management
The access management issues, which is exacerbated due to the lack of the joiner/mover/leaver process, can be addressed in multiple ways. We can provide reports on all the things an identity has access to, and whether some of that access is unnecessary since it was never used. Along with that, access reviews can be used to periodically require the owner of the identity to evaluate and assert that the identity and its access assignments are still neeeded. 
### Secure adaptive access
We need to provide capabilities that allow companies to assume breach and plan accordingly. Conditional access policies play an important role here, for example only allow a workload identity to authenticate from within my company’s network. With identity protection, we can analyze the access pattern of the past and check for anomalies in the pattern. This can then generate a risk score for reporting or even feed into the conditional access policies to block access when the risk is deemed high.
### Secrets management
It is best to avoid secrets in the first place if possible, and towards that we have made significant investments in “Managed identities” for Azure. Since the Azure platform manages the credentials, storing it securely and rotating it regularly, developers no longer need to deal with that pain. While this will avoid secrets in some scenarios, there are others where a secret is still necessary. So it becomes important to use vaulting solutions such as Azure KeyVault or other tools which can help with rotating these credentials and storing them securely.

## In conclusion
Recently we announced the public preview of access reviews for workload identities. And we will keep you updated as we round up the capabilities needed by customers to better manage, govern and secure workload identities.





