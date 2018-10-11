## Modern Service Mesh with Consul in Azure (Part 2)

This is the second part of a two-part series introducing you to HashiCorp Consul on Azure. In the [first part](https://open.microsoft.com/2018/10/04/use-case-modern-service-discovery-consul-azure-part-1/), we took a look at the service discovery properties of Consul and deployed a Consul cluster in Azure. In this second part, we will discuss properties that turn Consul into a full-blown service mesh solution as of version 1.2 and beyond. We will also look at how to make a variety of infrastructure services in Azure, including Kubernetes and PaaS, Consul and service mesh-aware.

### Service Mesh Properties in Consul
By definition, a service mesh solution should provide service discovery and runtime configuration management mechanisms for a set of microservices. In Part 1, we saw that Consul has been designed from the beginning to be a service discovery and service configuration management tool. What Consul did not have until version 1.2, however, was the option to transparently (i.e., without application code changes) authorize and secure communication between services. With the introduction of the Connect feature in Consul 1.2, the ability to secure services without making any assumption about the underlying networking infrastructure, completed the core feature set expected in a modern service mesh solution.

### Do you need Service Mesh?
Service mesh solutions have been getting a lot of attention as a necessary component of cloud-native infrastructure. And if you are deploying dozens or hundreds of microservices that need to discover and communicate with each other in a secure way, then service mesh is the right tool for that job. Many Azure customers, however, are simply moving monolithic applications into the cloud or developing applications that don't have the type of east-west (a convention used to refer to service-to-service communication) traffic patterns that service mesh was designed to help with. For the use cases where you have very few microservices present, you don't have to activate Connect properties of Consul; you can continue using Consul as a pure service discovery and configuration engine. 

On the other hand, if you are continuously adding new services into your application and have bumped into the realities of distributed computing at scale with its non-zero latencies, Consul service mesh will help you communicate over unreliable, unsecure networks with ease. Let's take a look at the architecture that enables Consul to accomplish that and go over configuration changes to enable Connect in the Consul cluster that we deployed in Part 1 of this series.

### Consul Connect Architecture - Proxies Everywhere
Consul Connect uses proxy sidecars to enable secure inbound and outbound communication without modifying services' code. In Part 1, you saw a basic Consul architecture diagram (cropped and pasted below). Note the proxy objects next to each application: those proxies verify and authorize the TLS connections, and then proxy traffic back to a standard TCP connection to the service itself. 

![Consul Architecture Cropped](https://github.com/echuvyrov/consul-part2/blob/master/architecture_cropped.png)

The application code for running services never becomes aware of the existence of Connect. Consul Connect can be enabled when the configuration of the Consul agent running that service includes a [special "connect" property](https://www.consul.io/intro/getting-started/connect.html). Within that property, the service can define upstream dependent services that it needs to communicate with over Connect, and all requests to those services will be authorized and go over secure TLS channels. 

For high performance services, Consul provides the option of [native application integration](https://www.consul.io/docs/connect/native.html) to reduce latency by avoiding the overhead of a proxy, but it does require some application code changes.

But how do we deploy Consul agents into various infrastructure environments in Azure?

### Deploying Consul Agents and Configuring Services
In Azure, there are three distinct infrastructure scenarios for deploying and configuring Consul agents. Let's go over each one of them in detail.

#### Consul Agents on top of IaaS
In the first scenario, Consul agents are being deployed on top of Virtual Machines or Virtual Machine Scale Sets in Azure. One approach for automating and easing configuration of Consul agents in Azure is illustrated below:


![Consul Agent IaaS Deployment](https://github.com/echuvyrov/consul-part2/blob/master/consulagent_iaas.png)

1. Service code and Consul agent configuration profile are stored in GitHub. DevOps engineers edit agent config file, perhaps enabling Connect or setting upstream dependencies.
2. GitHub check-in triggers an automated CI/CD build in Azure DevOps. Packer task or the newly announced [Image Builder Service](https://azure.microsoft.com/en-us/blog/announcing-private-preview-of-azure-vm-image-builder/) package the service, the Consul agent, as well as Consul agent config into a single VM managed image, ready to be deployed.
3. Terraform or ARM scripts reference the image created in Step 2, then add scripts to automatically start both Consul agent and service on boot.
4. VM is provisioned in Azure with Consul agent installed, configured for Consul Connect and with service running.

When configuration changes need to be made to the service, the workflow restarts with DevOps engineers editing config files in GitHub and triggering the build and redeploy steps.

#### Consul Agents on top of Kubernetes
In the second scenario, Consul agents are being deployed on top of Azure Kubernetes Service (AKS), a managed Kubernetes offer from Azure. In this case, DevOps engineers would use an official [Consul Helm chart](https://github.com/hashicorp/consul-helm) for deploying Consul servers and agents (or just agents in case we wanted to join the AKS cluster to the cluster we have already created):


![Consul Agent Kubernetes Deployment](https://github.com/echuvyrov/consul-part2/blob/master/consulagent_kubernetes.png)

1. Service Code and Helm chart with Consul agent configuration are stored in GitHub. DevOps engineers edit the Helm chart, enabling Connect or setting upstream dependencies.
2. GitHub checkin triggers and automated CI/CD build in Azure DevOps. Helm chart gets deployed using Azure CLI and/or a set of bash scripts.
3. Kubernetes cluster is provisioned in Azure with Consul agents installed, configured for Consul Connect and service running.

When configuration changes need to be made to the service, the workflow restarts with DevOps engineers editing Helm chart in GitHub and triggering the rolling update in AKS.

#### Consul Agents on top of Azure PaaS
In the third scenario, Consul needs to become aware of Platform as a Service Offering on Azure, such as Azure SQL Database. Those PaaS services are fully managed by Microsoft and expose a standard interface, against which a proxy service would need to be deployed and programmed to interact with.

![Consul Agent PaaS Deployment](https://github.com/echuvyrov/consul-part2/blob/master/consulagent_paas.png)

1. DevOps engineers create and deploy custom Consul Connect proxy that is able to create secure communication channels with PaaS services.
2. PaaS services interact with proxy and, as a result, become fully Connect-aware.

### Service Segmentation with Consul Connect
Now that you have both the Consul cluster and a set of services configured and running for Consul Connect, you can define security rules by specifying which services are allowed to talk to each other. Service identity is provided with TLS certificates that are generated and managed by Consul’s built-in certificate authority (CA) or other CA. This identity-based service authorization approach allows you to segment the network without relying on any networking middleware. You can accomplish this segmentation with [intentions](https://www.consul.io/docs/connect/intentions.html), which can be defined using the CLI, API or UI. For example, the following intention denies communication from db to web (connections will not be allowed).

```
$ consul intention create -deny db web
Created: db => web (deny)
```

### Happy Consuling
In Part 2 of this article series on Consul, you learned about service mesh properties of Consul, as well as several ways to deploy and configure it on Azure. When combined with service discovery and configuration properties, all easy to setup, use, and scale, Consul provides a powerful cloud-native solution for the microservices world.



