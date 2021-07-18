---
layout: post
current: post
navigation: True
title: Managing Workload Identities
date: 2021-06-22 07:00:00
class: post-template
subclass: 'post'
author: uday
---

## Introduction
Software workloads continue to move to the cloud as more businesses shift to cloud computing. Most new enterprise software developed these days are built natively to the cloud. These software workloads, made up of applications, scripts, services and daemons, typically need an identity to authenticate and access resources or communicate with other services. This has resulted in a dramatic increase in “workload identities” in cloud identity systems. 

Most cloud identity systems have a good level of maturity when it comes to securing, managing and governing user identities. However, this level of maturity does not currently exist for  workload identities. Their growing number along with the increasing awareness of the damage caused due to security breaches is causing many enterprises to pay closer attention to managing them in a better way. They are seeking capabilities such as lifecycle management, access management, secure adaptive access and secrets management. 

## Digging into the details...

### Terminology

First, let’s start with the terminology. There is no consistent terminology in the industry today for these “software identities”. In Azure AD, depending on what they are used for, they are called application identities, managed identities or service principals. In Windows Active Directory, there are service accounts which is a term that also shows up in many other platforms. The plethora of names leads to confusing outcomes especially when people sometimes use different terminologies to mean the same thing. 

Microsoft is using the term "workload identity" to refer to the identities needed by software workloads. Here are some sample scenarios that need workload identities: 
1. An application that is able to access Microsoft Graph based on admin or user consent, either on behalf of the user or on its own
2. A Managed identity used by a developer to allow a service to access an Azure resource such as Azure KeyVault or Azure Storage.
3. A Service prinicipal used by a developer to enable CI/CD with GitHub Actions.

“Workload identity” is a term that some industry analysts and product vendors are already using. To put these identities in perspective of the broader landscape of identities, it is useful to look at the taxonomy of all identities in Azure AD.

![Taxonomy of identities in Azure AD](/images/aad_taxonomy.png)

#### Human Identities 
These are the ones we are most familiar with. We use them ourselves every day, and companies use these identities to provision their employees and Frontline Workers (FLW) with access to things they need. These identities also allow external users, such as business partners or end consumers, to be provisioned with the right level of access.

#### Machine Identities (or non-human identities).

Machine identity is a term that is used by some industry analysts as well as product vendors to cover all the scenarios for non-human identities. We can classify them further into four broad categories. Over time, it is likely that these categories may overlap or even converge. 
- Mobile or Desktop Devices: Devices that we carry with us, either mobile or desktop, need an identity. In Azure AD, this identity is used to enable scenarios such as conditional access requiring a compliant device.  

- Internet of Things (IoT): These devices typically act on their own, unlike a handheld device or a desktop.

- Workload Identities: These are the identities referred to in this blog. They represent identities needed by applications, services and scripts to authenticate and access resources. 

- Robotic Process Automation (RPA) : RPA is a relatively new trend where human-based processes are being automated. Some of these scenarios end up needing a user identity, primarily to deal with how software is licensed. And in some other scenarios, they may use one of the different types of machine identities.


### So, what are some issues customers face with workload identities?

As the number of workload identities increases, identity administrators are starting to wonder how to manage their growing number. The regular spate of security breaches is causing a heightened awareness of the need to secure and govern these identities. Developers checking in the credentials/secrets for these identities in their code into their GitHub repos has been a common weakness that hackers are exploiting.  The most infamous of these exploits was with Uber several years ago. This is causing IT administrators to ask all the right questions: how do we avoid compromised workload identities trying to access our systems? Why can't we use conditional access policies as we do for users? What do we need to do to clean up these identities when they are no longer in use?

The nature of workload identities does not lend themselves to be managed in all the same ways we manage user identities. Let’s look at some of the unique differences to see why we need specialized capabilities for these identities. 

#### No joiner/mover/leaver process.
User identities have a clear process, typically called joiner/mover/leaver that is rooted in the HR system. When a user joins the company, they are added to the HR system which results in them being provisioned in the identity system. Using the information in the HR system, the user is also decorated with attributes such as their team and their role. This allows the user to be provisioned the right access needed for their job, for example by dynamically adding them to groups based on these attributes. When the user moves to a different team or role, they are automatically removed from the original group and placed in a different group, gaining access to new resources while losing access to the old ones. 

Workload identities don’t have a similar formal process that we can rely on. They get created by someone who deploys the application or service and are generally forgotten after that. It is usually difficult to figure out if a workload identity is still in use, and what it has access to. The person who created this identity may have forgotten why they created this, may have changed teams or even left the company. So then how do you manage the lifecycle of these identities? In most cases, this ambiguity leads them to remain in the system, forever untouched.

#### Need to be provisioned with credentials or secrets.
Unlike users who can typically show gestures or biometrics to prove who they are, workload identities need to be provided their credentials so they can use it to authenticate and get access to resources. Careful attention needs to be paid to manage these secrets. Many organizations have not built strong guidance in this area, and it is up to individual developers or devops team to figure this out. The rotation of secrets tends to be especially challenging since that part is hard for most folks to automate, and it tends to be done rarely and manually. Given this challenge, customers need better tools to ensure these credentials are stored securely and rotated regularly. And to deal with the possibility of security breaches, we also need capabilities such as anomaly detection in identity protection and conditional access policies such as only allowing access from certain networks.

#### Multiple identities with fine-grained access to a variety of resources
 A single workload may be configured with multiple workload identities. For example, the application identity may be used to access Microsoft Graph. And individual services within the application may have their own identities (aka service accounts), one of which may be provisioned to access Azure KeyVault and the other with access to Azure Storage. Given this characteristic, customers need ways to check what an identity has access to and how long that access should remain. 

### Improving the management of these identities

We hear from customers and industry analysts that this is a space that is fraught with problems primarily because there are no good solutions to tackle these issues. While niche tools do exist for things such as secret management and privileged access management, they solve a very small subset of the problem space. It is quite clear that customers will benefit from better ways to govern, secure and manage these identities. Here are thoughts on the core capabilities needed to address this area comprehensively.

#### Lifecycle management
The most important part of managing the lifecycle of these identities is to capture the meta-data on who created it and why it was created. This information needs to be automatically updated even as employees move teams or leave the organization. In addition, it is necessary to complement this information with data on the activity from that identity, such as when it last authenticated. 
#### Access governance
The access management issues, which are exacerbated due to the lack of the joiner/mover/leaver process, can be addressed in multiple ways. We can provide reports on all the things an identity has access to, and whether some of that access is unnecessary since it was never used. Along with that, access reviews can be used to periodically require the owner of the identity to evaluate and assert that the identity and its access assignments are still needed. 
#### Secure adaptive access
We need to provide capabilities that allow companies to assume breach and plan accordingly. Conditional access policies play an important role here, for example only allow a workload identity to authenticate from within my company’s network. With identity protection, we can analyze the access pattern of the past and check for anomalies in the pattern. This can then generate a risk score for reporting or even feed into the conditional access policies to block access when the risk is deemed high.
#### Secrets management
It is best to avoid secrets in the first place if possible, and towards that, Microsoft has made significant investments in “Managed identities” for Azure. The credentials for these identities are managed by the Azure platform, storing them securely and rotating regularly. While this will avoid secrets in some scenarios, there are others where a secret is still necessary. So it becomes important to use vaulting solutions such as Azure KeyVault or other tools which can help with rotating these credentials and storing them securely.

### In conclusion
Microsoft recently announced the [public preview of access reviews](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/introducing-azure-ad-access-reviews-for-service-principals/ba-p/1942488) for service principals. Expect to see more capabilities in this area for customers to better manage, govern and secure workload identities.

Comments/feedback/questions? [DM me on twitter](https://twitter.com/messages/compose?recipient_id=1446741344)