---
layout: post
current: post
navigation: True
title: Using Azure policy to control workload identity federation
date: 2022-10-28 10:00:00
class: post-template
subclass: 'post'
author: uday

---
In my [recent blog post]({% post_url 2022-10-03-azuread-federate-mi %}) we discussed the federation support on Azure managed identities. This capability allows you to use Kubernetes service accounts to access Azure resources without needing secrets. Developers can configure their managed identities with their cluster's OIDC issuer URL, adding trust between their Kubernetes service account and their managed identity. Since managed identities are Azure resources, you can use Azure policy to enforce rules like only using issuers trusted by your organization. This blog post walks through the use of Azure policy to achieve this control. 

 We will consider two scenarios in this blog post. Only allow federation on managed identities to an approved list of issuers and only allow federation with Azure Kubernetes Service (AKS) clusters in approved tenants.

## A quick primer: what is Azure policy?
 Azure policy is a service in Azure that allows you to define policies on Azure resources. It enables you to enforce business rules as well as check if resources are compliant with these rules. These rules apply to scopes, such as management groups, subscriptions, resource groups, or individual resources. 

If you want to author policy definitions that apply to all azure resources in your tenant, you can assign those definitions to the root management group. This group includes all management groups and all subscriptions in the tenant.

To work with Azure policies, [visit the Azure portal](https://portal.azure.com), search for "policy", and pick the "Policy" service. You will see an overview of Azure policy. You can view existing policy definitions and the compliance state of resources, define new policy definitions, and assign those definitions to a hierarchy of scopes.

Azure policy definitions use a JSON format to specify the rules that need to be evaluated by the policy. The policy definitions can get complicated and hard to debug, so be patient and experiment as you build your expertise in this area. There is also a Visual Studio Code extension for Azure policy, which simplifies this experience.

Azure evaluates policies and enforces them when your create new resources. Any existing resources that don’t adhere to the rules defined in the policy will show as non-compliance in the compliance report.

## Defining a policy to restrict federation with an approved set of issuers.
Let's see how we can define an Azure policy to allow only a known set of OIDC issuers.

In this example, assume you want to only allow three OIDC issuers for federation with managed identities in your tenant (or a specific management group or subscription).
- An EKS cluster in AWS
  > https://oidc.eks.us-east-2.amazonaws.com/id/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
- GitHub actions
  > https://token.actions.githubusercontent.com
- An AKS cluster in Azure 
  > https://westus3.oic.prod-aks.azure.com/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy/zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz/

Go to the authoring section of Azure policy in the portal, and pick "Policy definition".
- For the location, pick the management group or subscription where you want to define this policy.
Use a name that makes sense for you. For example: "Approved issuers for federated credentials".
- Description: "Only allow federated credentials which use an approved OIDC issuer".
- Category: either pick a category that already exists or create a new category (for example, Managed Identity Federated Credentials)

![Azure Policy definition](/images/mi-federate-policy/policydefine-1.jpg)

The policy rule is as follows:
```
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials"
        },
        {
          "not": {
            "value": "[field('Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/issuer')]",
            "in": "[parameters('allowedIssuers')]"
          }
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {
    "allowedIssuers": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed issuers",
        "description": "The list of allowed issuers for federated credentials."
      }
    }
  }
}
```
This policy definition says:
- when creating a resource of type federatedIdentityCredentials
- if the issuer is not in the allowedIssuers list
- deny the creation

We will be able to provide the list of approved issuers when we assign this policy.

Click on Assign policy and provide the scope and parameters for this policy.

![Azure Policy definition](/images/mi-federate-policy/policydefine-2.jpg)

Once the policy is assigned, Azure policy will check this policy against all existing resources of this type. In a few minutes, you will see a compliance report. In my case, it shows several resources are already out of compliance.

![Azure Compliance report](/images/mi-federate-policy/policycompliance-1.jpg)

Picking one of them provides details on why the resource is out of compliance.

![Azure Compliance report detail](/images/mi-federate-policy/policycompliance-2.jpg)

Now, when creating a federated identity credential on a managed identity that is in scope for this policy, the creation is allowed only if the issuer is in the allowedIssuers list. Otherwise, it will fail with an error. Unfortunately, the error message is very obscure and does not tell you anything specific to figure out what happened.

![Azure policy disallowed](/images/mi-federate-policy/azurepolicy-disallowed.jpg)


## Define a policy to allow multiple AKS clusters in a tenant
While the allowedIssuers policy that we just defined should meet most of your needs, in some cases, you may need to configure this list with tens or hundreds of Kubernetes clusters. Azure Kuberenetes Service (AKS) uses an issuer URL format that is policy friendly to meet our needs. The format of the AKS issuer URL is as follows: https://&lt;region&gt;.oic.prod-aks.azure.com/&lt;tenant-id&gt;/&lt;unique-id-for-cluster-url&gt;/
Knowing this format, we can author a policy definition that allows an approved issuer list or AKS clusters in a set of tenants.
```
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials"
        },
        {
          "not": {
            "anyOf": [
              {
                "value": "[field('Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/issuer')]",
                "in": "[parameters('allowedIssuers')]"
              },
              {
                "allOf": [
                  {
                    "value": "[if(greaterOrEquals(length(split(field('Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/issuer'),'/')),3),split(field('Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/issuer'),'/')[2],'')]",
                    "like": "*.oic.prod-aks.azure.com"
                  },
                  {
                    "value": "[if(greaterOrEquals(length(split(field('Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/issuer'),'/')),4),split(field('Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/issuer'),'/')[3],'')]",
                    "in": "[parameters('allowedTenants')]"
                  }
                ]
              }
            ]
          }
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {
    "allowedIssuers": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed issuers",
        "description": "The list of allowed issuers for federated credentials."
      }
    },
    "allowedTenants": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed tenants",
        "description": "Azure Kubernetes cluster issuers from these tenants are allowed".
      }
    }
  }
}

```
This policy definition says:
- when creating a resource of type federatedIdentityCredentials
- deny the creation unless the issuer is in the allowedIssuers list or the issuer is an AKS cluster in the allowedTenants list

I find the regular expression in Azure policy quite complicated. The greaterOrEquals check may not be necessary for this particular policy definition. It's there for completeness if you want to use a similar policy in other scenarios.


## In conclusion
You can use Azure policy to control which issuers are allowed on federated credentials for user-assigned managed identities. Getting the regular expressions in Azure policy can be a struggle, but it can achieve powerful outcomes! There are additional capabilities we have not discussed: for example, you can author exemptions to exclude certain resources from specific policies. You can also write policy definitions that are different for different resources, for example, fewer approved issuers for Azure resources in your production subscription. 

If you have any comments, feedback, or suggestions on this topic, I would love to hear from you. [DM me on Twitter](https://twitter.com/messages/compose?recipient_id=1446741344)
