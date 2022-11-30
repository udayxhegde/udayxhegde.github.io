---
layout: post
current: post
navigation: True
title: Service-to-service authentication and authorization in your Kubernetes deployments
date: 2022-05-05 10:00:00
class: post-template
subclass: 'post'
author: uday

---
As more applications move to the cloud, many development teams consider cloud-neutral and cloud-native technologies when architecting their deployments. Decomposing monolithic applications into multiple independent microservices can provide a simple and agile model. Kubernetes is gaining popularity as a cloud-neutral platform to deploy these microservices. This blog discusses how these individual microservices authenticate and authorize each other. It is different from the earlier blog posts on this site. Rather than a deep dive into a specific scenario or solution, I am soliciting your input on your identity needs for your software services.

With the workload identity federation, we tackled a specific scenario. It allows you to use an identity, such as a Kubernetes service account, for your service to access Microsoft cloud resources. There is a related scenario that I have been exploring recently. When multiple services need to communicate, a different set of capabilities are necessary. This blog post walks through some of the things I have learned. It's a start to a conversation on this topic. I need your help in filling gaps in my understanding.

By the way, I am visiting KubeCon in Valencia a few weeks from now. If you are attending the conference, and have any input on this topic, let's meet and chat! And if you are not going to KubeCon, let's connect virtually on this topic. [Connect with me on Twitter](https://twitter.com/messages/compose?recipient_id=1446741344)

## Identity needs for service-to-service communication
There appear to be three primary requirements that most developers care about: 
1. Mutually authenticate the interacting services. When Service A is communicating with Service B, it can verify it is indeed talking to service B and, vice versa, service B knows the communication is coming from service A. Why is this important? In the most common internet scenarios that we interact with every day, the HTTPS protocol ensures we talk to the intended website whose name we provide to the browser. While the server does not know the identity of the client, usually the browser, it has the means to authenticate the user interacting with the browser. When you sign-in, both sides of the communication are authenticated. The service-to-service communication needs a similar mechanism to authenticate the services to each other, even when there's no HTTPS or any user involved.

2. Authorize the access. When you have multiple services, each with different endpoints, it becomes necessary to set authorization rules for these individual endpoints. These rules control access to the service endpoints specifying what operations are permitted. Without these rules, you will not be able to detect when your services don't behave as unintended. Even beyond inadvertent access, authorization rules help guard against lateral attacks when one of the services is compromised.

3. Encryption: with the HTTPS protocol, all the communication between the client and server is encrypted. When services publish their endpoints over the internet, they generally use the HTTPS protocol. However, most internal endpoints are not  HTTPS endpoints. Many deployments use an unencrypted channel for this communication, especially when using a dedicated network. However, when these services are deployed in a public cloud, encrypting the communication between the services becomes necessary.

Three primary approaches seem to be prevalent in meeting these needs. They all rely on certificates as the underlying mechanism, enabling mTLS that ensures traffic encryption over the wire. 

1. Service-mesh. There are a variety of service meshes, each with a different flavor of how they meet the identity needs discussed above. While some people use service meshes primarily for their network traffic control, others rely on the security capabilities provided by the service mesh. The service mesh is a proxy-based approach, with the proxy handling all aspects of authentication, authorization, and traffic encryption. Developers don't need to build identity logic in their services.

From my limited knowledge of service meshes, here's an opinion I have formed:
Istio has the most sophisticated identity capabilities that work across multiple clusters. It uses SPIFFE under the covers. The SPIFFE identities are associated with the Kubernetes service accounts. Istio's citadel issues short-lived certificates to these identities, which can be rooted to your cert authority. Istio envoy proxy uses these certs for mutual authentication and traffic encryption. Istio also offers granular authorization policies. For example, you can configure service A to be allowed to POST at URL X on Service B, while Service C can only do a GET. Istio even allows you to plug in your OPA server for the authorization part. Some experiments blogged over the internet discuss how you can plug your SPIFFE server into Istio.

LinkerD, Consul, Open Service Mesh, and many other service meshes seem to offer a varying degree of identity integration for service to service needs.

The pros of this service-mesh approach:
- Developers don't need to handle any identity specifics in their code.
- It is unnecessary to update the service with new versions of identity libraries as they evolve, making it easier for developers.
- Open-source makes it easy to experiment and use without any costs involved.

The cons:
- The added latency of proxies, which I don't think are significant, but certainly seem to be a concern for some deployments
- The expertise needed to manage a service mesh deployment. This expertise seems to vary between the different service meshes. Istio appears to be the most complex, sometimes needing separate teams that manage the Istio deployments. 
- Needing to deploy open-source components, which do not come with any SLA or support. 

2. Deploy open-source authentication and authorization services on your own
In this approach, developers use open-source solutions that address their need for service-to-service authentication, authorization, and encryption. SPIFFE offers an attractive option to provide services with an identity. The short-lived certificates provided to these identities can encrypt the network traffic using mTLS. In addition, the SPIFFE architecture offers a rich and flexible model. Node attestors offered by SPIRE, the open-source implementation of SPIFFE, provide a variety of means for attesting nodes added to the trust fabric. 
Workload attestors can evaluate several workload characteristics to determine the identity assigned to that workload. These characteristics can range from the Kubernetes service account used by the service to an image signature that forms a thumbprint of the service. SPIFFE does not offer any authorization support. OPA (Open Policy Agent) is often used to complement a SPIFFE deployment for authorization needs.

The SPIFFE and OPA architecture is flexible and can work in different deployment options. Services can be deployed across multiple clusters, in multiple clouds, and even mixed with non-cluster deployments.

Beyond SPIFFE and OPA, there are probably other equally viable options.
 
The pros of this approach:
- Consistent way to achieve authentication, authorization, and traffic encryption across a variety of platforms and cloud deployments
- Standards-based approach, so you are not stuck with a specific implementation
- Open-source makes it easy to experiment and use without any costs involved.

The cons:
- Developers' code needs to call the APIs provided by these services. Any update to these APIs will need code changes. 
- Needing to deploy open-source components, which do not come with any SLA or support.
- All aspects of scale (such as multi-cluster support) and disaster recovery need to be handled.


3) Build your custom logic for authentication and authorization
In this approach, developers build custom logic for generating certs for each service. These certs help achieve mutual authentication and traffic encryption. Authorization is additional custom logic needed in this flow.

There are several security and hygiene considerations that need to be considered when building this custom logic. These include using a PKI versus self-signed certificates, how long certs are valid, and the logic to automate cert rotation.

The pros of this approach:
- I am not sure of all the pros here. I can imagine this may be easier to deploy, with a lot more control on how things work end-to-end?

The cons of this approach:
- Seems a lot more complex when you want to scale this out to a large set of services
- Needs careful planning to ensure a secure deployment

### Some additional thoughts and caveats
- Some deployments seem to depend on capabilities available within a single Kubernetes cluster, such as the service account token or self-signed certs. However, this approach can be limiting when the service needs to scale and be deployed across cluster boundaries. I think the ability to scale should be designed from day 1 to avoid rearchitecting this solution later on.
- Having a solution that works beyond Kubernetes deployments: while Kubernetes provides a consistent platform across cloud providers, some deployments have additional needs, such as allowing services to run on bare-metal or VMs. 
- Managed solution: All the options we have discussed entail DevOps or DevSecOps deploying and managing the identity aspects themselves. A managed identity solution does not seem to be available.

### In conclusion
This blog post attempts to articulate the available options with their pros and cons. These are all opinions I have formed in my own evaluation. I know I could be wrong on several fronts and missing additional detail. 

Is this an important problem that you deal with today? What additional requirements do you have in this area? I am curious to understand and would love to hear from you. Please [DM me on Twitter](https://twitter.com/messages/compose?recipient_id=1446741344)
And, if you are attending KubeCon in Valencia in May, let's meet up!


