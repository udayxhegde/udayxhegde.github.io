---
layout: post
current: post
navigation: True
title: Access AWS resources using Azure AD identities
date: 2021-09-14 07:00:00
class: post-template
subclass: 'post'
author: uday
---
Azure AD service principals can be used in scenarios where you need non-interactive, programmatic access to AWS resources. While AWS documentation explains how this can be achieved for users from certain identity providers, it is lacking concrete examples for achieving this using service principals in Azure AD. This blog post provides a walkthrough of how this can be achieved.

## Introduction
Amazon Web Services (AWS) has support and documentation for federating users from any identity provider, and granting them access to AWS resources using [AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html). AWS also has  related documentation on [the general pattern of web identity federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc.html) and [the available keys for AWS web identity federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_iam-condition-keys.html#condition-keys-wif).

Developers who need to build services that access AWS resources, such as S3 or DyanmoDB, can achieve this scenario using Azure AD identities and without needing to manage any credentials. They can assign a managed identity to their service running in Azure and use the Azure AD tokens issued to the managed identity to access AWS resources.

## First create a managed identity in your Azure subscription
We will look at other ways you can use Azure AD workload identities to federate with AWS later in this blog. For now, let's look at the simplest way to achieve this using a managed identity in Azure AD.

Go to the Azure portal and create a "User assigned managed identity", and copy its client id and object id, you will need these later.
![Adding a managed identity](/images/aad_aws_federate/aadmanagedidentity_img.jpg)

## Setting up AWS federation with your Azure AD Tenant
### Create an identity provider in AWS
Head over to the AWS console, and select the IAM service. Click on "Identity providers", and "Add provider". Choose the "OpenID Connect" option.
In the provider URL, enter https://sts.windows.net/[your-tenant-id]. If you dont have your tenant id handy, see [how to find your tenant id](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-how-to-find-tenant).


Click on "Get Thumbprint". I am not sure why AWS is acquiring this statically. I would have expected AWS to follow the chain of trust dynamically as needed.
![Add an identity provider](/images/aad_aws_federate/awsidentityprovider_img.jpg)

Configure the audience using the client id of your managed identity and click on "Add Provider".
![Configure with audience](/images/aad_aws_federate/awsidentityprovider1_img.jpg)

### Assign a role to that identity provider
Next, you need to create an AWS role and assign it to this provider. This will allow you to use the Azure AD tokens to assume this AWS role. Click on the identity provider you just created, and click on "assign role". 

In "Select type of trusted identity", select "Web identity". In Choose a web identity provider, AWS would have already populated the identity provider, confirm it is the right one. And select the audience you just configured on the provider. (Note: You can configure this identity provider with additional audiences if you need to.)
![Assign role to identity provider](/images/aad_aws_federate/awsidentityprovider3_img.jpg)

Click on permissions and configure the finer-grained permissions you want to grant to this role. You can pick among existing policies or create a custom policy. In the review step, you get to provide a name for this role and a description, and click "Create Role".

Now, pick the role you just created, click "trust relations", and "Edit trust relationship".

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<your-aws-account-id>:oidc-provider/sts.windows.net/<your-tenant-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "sts.windows.net/<your-tenant-id>:aud": "<client id of managed identity>"
        }
      }
    }
  ]
}
```
Anyone who can get a token for this audience in your tenant will be able to access the AWS resources granted to this role. So you should restrict this further to the specific identity you are allowing, by using the subject condition.


```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<your-aws-account-id>:oidc-provider/sts.windows.net/<your-tenant-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "sts.windows.net/<your-tenant-id>:aud": "<client id of managed identity>",
          "sts.windows.net/<your-tenant-id>:sub": "<object id of managed identity>"
        }
      }
    }
  ]
}
```
## Finally, access the AWS resources from your service!
You have now completed all the configurations needed to try this out end to end.
Let's look at a code sample to get Azure AD tokens and provide that to AWS.

Assign the managed identity to your Azure App Service (or Functions) where your code is running. Then use the Azure Identity SDK to get a token for that managed identity with the audience being the client id of your managed identity. Here's a code snippet using Node.js
```javascript
    const identity = require("@azure/identity");
    const clientId = "<client id of managed identity>"
    const audience=clientId;

    const credential = new identity.DefaultAzureCredential(
                    { ManagedIdentityClientId : clientId });
    const token = credential.getToken(audience);
```

Note that this is the OAuth Confidential Client flow: you get access tokens, but no refresh tokens. You need to make sure this token is refreshed regularly before it expires. The instructions for how to do this vary depending on the AWS SDK you use.
For example, when using the aws-sdk for Node.js, here's a snippet I used
~~~javascript
public tokenRefresh = async() =>{
    return credential.getToken(audience)
    .then ((response:any) => {
        if (response) {
            //update the token in the AWS credentials class
            AWS.config.credentials.params.WebIdentityToken = response.token
            //set a timer to refresh the token when it expires. We may need the timer to trigger
            //a bit earlier, and not wait till the token is expired
            setTimeout(tokenRefresh, 
                       response.expiresOnTimestamp - Date.Now());
        }
        else {...}
    }
    .catch(...){...}
}
~~~
## Things to keep in mind
Since you are granting permissions to your AWS resources, you need to be very careful in your configuration so that only the identities you are expecting can get this access.

You can use two keys available in the AWS web identity federation to configure this properly, the "aud" key and the "sub" key.

The aud refers to the audience in the Azure AD token. Note that several identities in your tenant will likely be able to get a token for the audience you picked. So we used the "sub" condition key to restrict this to only tokens issued to our managed identity.

## Alternate route, using an Azure AD Application.
You can also use an Azure AD application and use the Azure AD App model capabilities for some additional flexibility: especially in being able to map multiple managed identities to the AWS role.

Configure your Azure AD application to require user assignment, create a role on the app, and then assign your selected managed identities to that role. This will allow only identities assigned to the app to get a token with this audience. However note that AWS does not support access enforcement using the role claim provided by Azure AD in the token. So you need to be careful, since assigning identities to any other roles on that application will also allow them to assume the AWS role.

Here's the configuration information if you choose to go this route.
Go to your Azure AD portal, and visit Application Registrations, and create an application. Configure the Application ID URI.
![Create an Azure AD application](/images/aad_aws_federate/aadapplication_img.jpg)

The URI can be any name you choose, and it will be the audience when you acquire a token for the app. For example: api://AWS_Application or URN://AzureAD_AWS_Federate
![Configure with app URI](/images/aad_aws_federate/aadapplication1_img.jpg)
Next, create an application role, and specify that only applications are the allowed member types. This part can be a bit confusing, since "applications" here is a misnomer. It should really say "Service Principals", which also includes managed identities.
![Create app role](/images/aad_aws_federate/aadapplication2_img.jpg)

Then, configure the application to require user assignment. Again this part can be confusing, since it is the "user assignment" is a misnomer, and you can assign any Azure AD identities, including service principals. This step will ensure that only those identities assigned to the application can get a token for the application audience.
![require assignment](/images/aad_aws_federate/aadapplication3_img.jpg)

Finally, we need to assign our managed identity to this role. Unfortunately, Azure AD does not provide this role assignment in the portal. You have to use PowerShell, and bear with me since this PowerShell cmdlet is quite a handful with duplicate parameters which should have been unnecessary. This document [shows how to assign a managed identity to an app role](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-to-assign-app-role-managed-identity-powershell)

For this step you need 3 things: the object id of the managed identity, the identity of the app role you created, and the object id of the service principal that represents the application you created. You can find this last part by going to the Enterprise Applications on the Azure AD portal, and search for the application name you just created. The object you find is the service principal of the application, and you can look up the object id of that service principal.

You will be executing a PowerShell cmdlet that looks like this:
```
New-AzureADServiceAppRoleAssignment -ObjectId <object id of your managed identity> -Id <identity of the app role> -PrincipalId <object id of managed identity again!> -ResourceId <object id of the service principal representing the application in your tenant>
```

Now configure the AWS role appropriately as follows:
Set your app URI audience on your identity provider in AWS. On the AWS portal, go to the IAM service, and pick identity providers. Select the identity provider you added and pick "add audience" under Audiences. Add the app URI of your app as the audience.

Then pick roles on the IAM service portal, and select the role we configured called AzureADWebIdentity. In the "Trust Relationships", edit trust relationship, and change the audience to the app uri of your application. Since you are already managing who can get the app tokens using the app role assignments on the application, the subject claim is optional and not needed. And especially if you have multiple managed identities assigned to the role, you will not be able to use the "sub" key since it will only allow you to specify one identity.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<your-aws-account-id>:oidc-provider/sts.windows.net/<your-tenant-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "sts.windows.net/<your-tenant-id>:aud": "urn://AzureAD_AWS_Federate",
        }
      }
    }
  ]
}
```

Change your code to specify your app uri as the audience when acquiring an Azure AD token.
```javascript
    const identity = require("@azure/identity");
    const audience="urn://AzureAD_AWS_Federate";

    const credential = new identity.DefaultAzureCredential(
                    { ManagedIdentityClientId : this.clientId });
    const token = credential.getToken(audience);
```


Hope this blog post makes it easier for you to setup access to AWS resources using Azure AD workload identities!

Comments/feedback/questions? [DM me on twitter](https://twitter.com/messages/compose?recipient_id=1446741344)
