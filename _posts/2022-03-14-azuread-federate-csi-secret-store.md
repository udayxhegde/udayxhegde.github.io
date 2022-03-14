---
layout: post
current: post
navigation: True
title: Azure KeyVault CSI secrets provider with workload identity federation
date: 2022-03-14 10:00:00
class: post-template
subclass: 'post'
author: uday

---

In my [earlier blog post]({% post_url 2022-01-11-azuread-federate-k8s %}) we discussed how to use the preview feature of Azure AD workload identity federation in Kubernetes clusters. You can use Kubernetes service accounts to access Azure resources without storing secrets for Azure AD workload identities. This capability is now available for Secrets Store CSI drivers as well! And it makes it easier to access your Azure KeyVault secrets securely from your Kubernetes deployments.

# What is Secrets Store CSI driver?
If you are as new to the Secrets Store CSI concept as I am, here's a brief intro to aid in understanding the contents of this blog. CSI (Container Store interface) enables a plugin architecture, to make new types of storage available to containerized workloads in Kubernetes clusters. The Kubernetes Secrets Store CSI driver uses this model to allow a variety of secret store providers to expose their secrets in the form of files to pods running in the cluster. You don't need to write custom code to connect to a specific secret manager and read from it. Your code is easier to write and doesn't need to change when you move to a different secret store.

Several providers are available for Secrets Store CSI drive, connecting to secret stores. These include AWS Secrets Manager, Google Secret Manager, Azure KeyVault, and HashiCorp Vault. Each provider deals with the specific details to connect to their respective secret manager, so you don’t have to put all of that logic in your code.

# Updated Azure KevVault CSI secrets store provider
Now we have a basic understanding of Secret Store CSI Driver, let's take a look at the recent changes to the Azure KeyVault provider. It's updated to use the new workload identity federation mechanism available in Azure AD. With this update, you can securely access Azure KeyVault secrets from a Kubernetes cluster, no matter where it is deployed.

This blog post will first walk through using this new capability in your Kubernetes cluster. Later in the blog post, we will compare this model with the other authentication options supported by the Azure KeyVault provider.

## Part I: Using the new provider using the federated flow

You need a Kubernetes cluster that has enabled an OIDC issuer. The instructions to achieve this vary depending on your flavor of the Kubernetes cluster. See [this walkthrough]({% post_url 2022-01-11-azuread-federate-k8s %}#AKS-OIDC) in our earlier blog on how to do this for Azure managed clusters.

The mutating admission webhook, discussed in the earlier blog post, is not needed here. It will not be used in this flow even if it is installed in your cluster. The Azure KeyVault CSI provider takes care of all the necessary config: so all you need here is a cluster that is OIDC enabled. 

Here are the steps to try out the updated Azure KeyVault CSI provider. 

### install the updated Azure KeyVault secrets store provider 

The Azure KeyVault CSI secrets provider has [been updated](https://github.com/Azure/secrets-store-csi-driver-provider-azure/pull/778) with the workload identity federation capability. 

Follow the [instructions](https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/getting-started/installation/)
to install these prerequisites on your cluster:
- Secrets Store CSI Driver v1.1.0+
- Azure Key Vault Provider v1.1.0+

Once you have these out the way, the rest is simple:

### Add a Kubernetes service account in your deployment
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workload-identity-sa
---
```

### Configure the Azure AD application to trust the service account
Add a federated identity credential on the Azure AD application to which you have granted access to the secrets in your KeyVault. This allows the CSI driver to exchange the Kubernetes issued tokens for Azure AD access tokens needed to access Azure KeyVault. For this step, you need the issuer URL for your Kubernetes cluster. Each flavor of the Kubernetes Cluster, such as EKS, GKE, and AKS, will provide instructions for getting this information. In the case of AKS, you can get the issuer URL doing the following:

```
az aks show --subscription $SUBSCRIPTION \
         -g $RESOURCE_GROUP --name "${CLUSTER_NAME}" \
         --query 'oidcIssuerProfile.issuerUrl' -otsv
```

Using the issuer URL, the namespace of your deployment, and your service account, you can head to the Azure AD portal to configure the federated identity credential.

See [how to achieve this]({% post_url 2022-01-11-azuread-federate-k8s %}#AADPORTALFIC), described in the earlier blog.

### Add a SecretProviderClass to your deployment. 

You will provide the clientID of your Azure AD application, along with the tenantID and your Azure KeyVault instance in this configuration.

```
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-secrets
spec:
  provider: azure
  parameters:
    clientID: "aead113c-24c6-47e3-93ce-3a7dafc5fd99"
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    keyvaultName: "codesamples-kv"
    objects: |
      array:
        - |
          objectName: demo-secret
          objectType: secret
          objectVersion: ""
    tenantId: "72f988bf-86f1-41af-91ab-2d7cd011db47"
---
```

The objectName is a secret in my Azure KeyVault instance. We will access that secret from our pod deployment.

### Deploy your pod to use the service account and SecretProviderClass
Now in your Pod deployment, you refer to the service account and the secret class provider:

```
kind: Pod
apiVersion: v1
metadata:
  name: csi-secret-pod
spec:
  serviceAccountName: workload-identity-sa
  containers:
    - name: azure-csi-secret
      image: acruday.azurecr.io/azurecsisecret:v6
      volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-secrets"

```

### Access your secret from your pod
At this point, the secrets you have picked in the secret class provider will be available to the pod under /mnt/secrets-store. In my example, my pod will be able to access the secret called demo-secret in my Azure KevVault instance at /mnt/secrets-store/demo-secret!


## Part II: Comparing the authentication options for Azure KeyVault provider

Now, let's compare this with the other authentication options available in the Azure KeyVault CSI secrets store provider. Here are all the options:
- Using Service Principal secrets
- User assigned managed identity
- System assigned managed identity
- Pod identity
- Workload identity: this is the new option we discussed earlier in the blog

Here is a summary of these options that also shows their pros and cons:

### Service Principal secret
In this option, you create a secret on the Azure AD application and store it as a Kubernetes secret. You can then refer to that secret when you mount the CSI volume. The danger of using secrets is hopefully obvious. If the secrets were to leak, your Azure KeyVault secrets are compromised. Another problem with this approach is that someday this secret will expire. Suddenly, your pod will experience downtime when accessing the KeyVault secrets.
#### Pros of this approach: 
- Works no matter where the cluster is deployed, in Azure or elsewhere.

#### Cons:
- It is fragile. The service principal secret can leak, causing a security risk. They expire, leading to downtime if you are not careful in being proactive in handling the expire

### User-assigned managed identity
When your cluster is deployed in Azure, you can use a user-assigned managed identity. You have to assign this managed identity to VMSS or each VM belonging to the cluster. You have to grant the user-assigned managed identity access to your KeyVault. Then, in the SecretProviderClass, you do the following:
```
	useVMManagedIdentity: "true"
  userAssignedIdentityID: "<client id of the managed identity>"
```

Please don't use this option since it is not secure. Not only can your pod in your namespace access your KeyVault secrets, so can every other pod in every namespace within that cluster!!! Since the managed identity is assigned to each node, all pods in the cluster can get tokens for that user-assigned managed identity and access your KeyVault. My recommendation: **Not secure. Do not use this option**.

#### Pros
- Simple to use

#### Cons
- Need cluster-admin privileges to deploy managed identity on all nodes in the cluster
- All pods in the cluster can access your KeyVault secrets, which is a security risk

### System-assigned managed identity
This option is similar to the user-assigned managed identity option. Your cluster is probably already using a system-assigned managed identity: why not use that same identity? You grant the system-assigned managed identity access to your KeyVault. And tell the SecretClassProvider to use this option:
```
	useVMManagedIdentity: "true"
```

#### Similar pros and cons as user-assigned managed identity
Recommendation: **Not secure. Do not use this option**.


### Pod identity
The [pod identity project](https://github.com/Azure/aad-pod-identity) addresses many security issues in the other options discussed above. It requires you to deploy the pod identity components and use the CRDs, so that only the appropriate pods can get the tokens for the managed identities.

#### Pros
- More secure than the earlier options

#### Cons
- only works on clusters deployed in Azure
- only supported on Linux
- needs installation of pod identity components in your cluster
- requires cluster-admin support, since managed identities have to be assigned to each node in the cluster

### Workload Identity
This option is the most secure among all the options discussed above. It is also native to Kubernetes, without needing the addition of CRDs. 

#### Pros
- works on clusters deployed anywhere
- supports Windows and Linux nodes
- cluster-admin does not need to handhold each deployment
- uses native Kubernetes concepts of OIDC federation and service account tokens

#### Cons
- Currently in preview, not generally available
- only available for Azure AD applications, managed identity support coming soon

### In conclusion
Use the workload identity option in the Azure KeyVault CSI secrets store provider. It is the more secure and flexible option, though still in preview. Stay tuned for the managed identity support coming soon in Azure AD workload identity federation!

If you have any comments, feedback, or suggestions on this topic, I would love to hear from you. [DM me on Twitter](https://twitter.com/messages/compose?recipient_id=1446741344)
