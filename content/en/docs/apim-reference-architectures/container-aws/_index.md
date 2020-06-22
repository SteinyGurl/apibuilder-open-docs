
{
"title": "Axway AMPLIFY™ API Management Container Reference Architecture on AWS",
"linkTitle": "API Management Container Reference Architecture on AWS",
"weight": 30,
"date": "2019-12-20",
"description": "Reference Architecture"
}

Table of Contents
=================

[1. Overview](#overview)  
[2. General architecture](#general-architecture)  
[2.1. Principles](#principles)  
[2.2. Target use case](#target-use-case)  
[2.3. Additional components and considerations](#additional-components-and-considerations)  
[2.3.1. Container registry](#container-registry)  
[2.3.2. Bastion host](#bastion-host)  
[2.3.3. DevOps Pipeline](#devops-pipeline)  
[2.3.4. SMTP Relay](#smtp-relay)  
[2.4. Performance goals](#performance-goals)  
[3. Implementation details](#implementation-details)  
[3.1. Diagram](#diagram)  
[3.2. Choice of runtime infrastructure components](#choice-of-runtime-infrastructure-components)  
[3.2.1. VM sizes](#vm-sizes)  
[3.2.2. Storage](#storage)  
[3.2.3. Load balancer](#load-balancer)  
[3.2.4. Network](#network)  
[3.3. Kubernetes considerations](#kubernetes-considerations)  
[3.3.1. Deployment options](#deployment-options)  
[3.3.2. Namespaces](#namespaces)  
[3.3.3. Pod resource limits](#pod-resource-limits)  
[3.3.4. Components healthcheck](#components-healthcheck)  
[3.3.5. Affinity and anti-affinity mode](#affinity-and-anti-affinity-mode)  
[3.3.6. Autoscaling](#autoscaling)  
[3.3.7. External traffic](#external-traffic)  
[3.3.8. Secrets](#secrets)  
[3.4. API Management implementation details](#api-management-implementation-details)  
[3.4.1. Admin Node Manager](#admin-node-manager)  
[3.4.2. API Manager UI](#api-manager-ui)  
[3.4.3. API Gateway/Manager (traffic)](#api-gatewaymanager-traffic)  
[3.5. Cassandra considerations](#cassandra-considerations)  
[3.6. Security considerations](#_Toc27752104)  
[3.7. SQL database considerations](#sql-database-considerations)  
[3.8. Logging/tracing](#loggingtracing)  
[3.9. Environmentalization and Promotion](#environmentalization-and-promotion)  
[3.10. Infrastructure components for AWS deployment](#infrastructure-components-for-aws-deployment)  
[3.11. Performance testing](#performance-testing)  
[4. Maintenance](#maintenance)  
[4.1. New configurations](#new-configurations)  
[4.2. Product updates](#product-updates)  
[4.2.1. Installing a patch](#installing-a-patch)  
[4.2.2. Installing a service pack](#installing-a-service-pack)  
[4.2.3. Upgrading the product](#upgrading-the-product)  
[4.2.4. Adding customization](#adding-customization)  
[4.3. Pushing a new Docker image to your Kubernetes cluster](#pushing-a-new-docker-image-to-your-kubernetes-cluster)  
[5. Configuration and data backups](#configuration-and-data-backups)  
[6. Disaster recovery](#disaster-recovery)  
[7. Known constraints and roadmap](#known-constraints-and-roadmap)  
[8. Appendix A -- Glossary of Terms](#appendix-a-glossary-of-terms)

Summary
=======
This document provides a reference architecture guide for deploying
AMPLIFY API Management (APIM) using Externally Managed Topology ([EMT
mode](https://docs.axway.com/bundle/axway-open-docs/page/docs/apim_installation/apigw_containers/container_getstarted/index.html)).
Deploying APIM using Docker containers orchestrated by Kubernetes brings
tremendous benefits in installing, developing and operating an API
management solution.

This document describes all major areas in deploying and maintaining
Axway APIM EMT on Amazon AWS cloud, including:

-   Physical and deployment architectures

-   Explanation and consideration for selecting underlying
infrastructure components

-   Kubernetes considerations

-   Performance, logging and monitoring aspects

-   Backup and recovery, including disaster recovery

-   Constraints and roadmap

Overview
========

AMPLIFY™ API Management is a leading API management solution on the
market. It supports container-based deployment under an option called
Externally Managed Topology (EMT). The purpose of this document is to
share Axway reference architecture for the container-based deployment of
an API management solution on Kubernetes. It will address many
architectural, development and operational aspects of the proposed
architecture.

Since the technology choices, Docker and Kubernetes, are portable across
on-premises environments and many cloud providers, most of the
information in this guide should apply to those environments. But we
include specific recommendations for AWS as one of the most common
deployment targets.

The target audience for the document is architects, developers, and
operations personnel. To get the most value from this document, a reader
should have a good knowledge of Docker, Kubernetes, and API management.

General architecture
====================

This chapter is focused on general architecture in support of an API
management deployment on a dedicated Kubernetes cluster. The chapter
discusses architectural principles, as well as required and optional
components. There are many ways to deploy software on a Kubernetes
cluster, but this document shares Axway's experience acquired from
deploying AMPLIFY API Management in an actual production environment.
Most of the implementation details will be outlined in the following
chapters.

Make sure the constraints listed in the chapters are respected in case
of deployment on an existing Kubernetes cluster.

Principles
----------

The name of the new deployment option --- EMT --- gives a good clue that
with this option, many operational aspects of the architecture are
externalized to an orchestration component. Existing users of AMPLIFY
API Management should be aware that with EMT deployment, the role of
Admin Node Manager becomes more of a monitoring tool. And **Node
Manager** is completely removed from the EMT architecture.

Official testing is taking place in Kubernetes as the orchestration
component. However, the Docker images are agnostic, so they can be
deployed in other orchestration platforms, like Swarm. Kubernetes
manages many important aspects of runtime, security, and operations:

-   Autoscaling API Gateway with CPU or memory consumption threshold

-   Self-healing

-   Rolling updates with zero downtime

-   And many more

In generic terms, reference architecture can be built by stacking four
layers of different capabilities:

![](/Images/apim-reference-architectures/container-aws/image1.png)

Notice that the packaging of API Management for deployment has changed
in the EMT mode. Customers need to build a new Docker image for any new
FED or POL file that they want to deploy (see
[documentation](https://docs.axway.com/bundle/APIGateway_77_ContainerGuide_allOS_en_HTML5/page/Content/ContainerTopics/containers_docker_setup.htm)).
We recommend creating a DevOps pipeline that can be triggered anytime a
new configuration is ready for deployment.

To manage Docker images, customers need to set up a Docker registry as a
repository for created images. To help customers with setting up a
required environment, the following table describes the required and
recommended options.

|Description|Required?|
|----|----|
|A DevOps pipeline is strongly recommended for building Docker images                                          |Recommended|
|A storage system is required with the capacity to store dedicated data and to share data between components   |Required|
|A bastion is required for administration tasks on API management and Kubernetes                               |Required|
|Kubernetes must have a container registry to pull Docker images.                                              |Required|
*Table 1: Deployment architecture - global recommendations*

Besides [Docker](https://www.docker.com/resources/what-container) and
[Kubernetes](https://kubernetes.io), we use [Helm](https://helm.sh)
scharts to describe the entire deployment configuration. Using Helm
provides an efficient way to package all configuration parameters for
Kubernetes and API Management containers to be deployed with a single
command.

Target use case
---------------

We review a single deployment option where all API Management components
are running inside a single Kubernetes cluster. This pattern allows one
to virtualize back-end apps or APIs hosted inside or outside the
cluster. It's possible to deploy multiple API gateway groups in the same
cluster. But keep in mind that every API gateway group configuration
should be deployed as a separate Docker image.

In our scenario, we expose several entry points for the external clients
inside the cluster (see implementation details in the next chapter). All
API gateways write their transaction events to a shared volume. Admin
Node Manager streams those events to a relational database management
system. API Gateway Manager UI can be used to view the traffic monitor
and events data.

A dedicated deployment environment requires:
-   Kubernetes cluster
-   Docker registry
-   Cassandra cluster
-   RDBMS
-   Storage system for shared volume
-   Monitoring system
-   Bastion system for cluster access
-   DevOps pipeline(s) to control Docker image build and deployment to a target environment

The following diagram shows a general architecture of a single cluster
configuration:

![](/Images/apim-reference-architectures/container-aws/image2.jpg)

Additional components and considerations
----------------------------------------

In this section, we present considerations for using additional
components in the single cluster architecture.

### Container registry

This registry is used to store Docker images and Helm packages. You may
consider maintaining at least two pipelines: one for building images and
the other for deploying them to a target environment. The image building
pipeline would build and push images to the registry. The deployment
pipeline would deploy a specific Helm package. In this scenario, Docker
images are pulled by Kubernetes and deployed to a cluster. One of the
requirements in this environment is to securely pass container registry
credentials during Helm package deployment. We suggest using Kubernetes
secrets. Besides registry access credentials, you will need to maintain
additional sensitive data that is described in the following table.

|Description|Required?|
|----|----|
|Docker images contain such sensitive data as license key, certificate, and configuration. This data must be protected. |Required|
|The password is sensitive and must be encrypted in the system.|Required|
|Operations should define a clear tag strategy for Docker images tagging.|Recommended|

### Bastion host

Administrative tasks should be executed safely. We use a bastion host to
bridge to the following instances via the internet:

-   Kubernetes master nodes for managing a cluster
-   Kubernetes Dashboard
-   RBDMS and Cassandra
-   Debugging any issue with a Kubernetes cluster

The bastion must have high traceability with specific RBAC permissions
to allow a few selected users to access infrastructure components.

### DevOps Pipeline

A DevOps pipeline is based on a suite of tools to build and push gateway
configuration and APIs to a target deployment environment and run a set
of verification tests. Since a Docker container is immutable, any change
in API gateway configuration requires building a new Docker image.

Axway provides several CLI tools that fit nicely into a DevOps pipeline.
Later in the document, we will describe:

-   Docker build scripts
-   API promotion tool (*apimanager-promote*)
-   GitHub projects

### SMTP Relay

The API Manager component can send alerts and messages. By default, the
solution only supports SMPT(S) to send an email. But customization can
be done in alerts policies to send an email by API.

Performance goals
-----------------

An important factor for achieving your goals with the EMT deployment is
to define a set of performance goals. These will be unique for a
specific set of APIs, deployment platform and clients' expectations.
Later in the document, we show an example of the performance metrics
that have been achieved in testing a reference architecture by the Axway
Managed Cloud team.

Implementation details
======================

This chapter details the configuration for each component.

Diagram
-------

![](/Images/apim-reference-architectures/container-aws/image3.png)

This figure shows the recommended reference
architecture diagram. It is designed with High Availability (HA) in
mind. An HA deployment requires redundancy and high throughput for all
infrastructure components and networks. To reach this target, components
must be deployed in multiple zones. In our configuration, we use three
availability zones. This configuration is compliant with a minimal
technical SLA of 99.99 percent. The implementation details of this
diagram are explained in this chapter.

Choice of runtime infrastructure components
-------------------------------------------

This section provides recommendations for a typical implementation of
the runtime infrastructure components.

In this configuration, all assets of the Kubernetes cluster are deployed
in the same data center or a region (in case of a cloud deployment),
although components are spread out in various racks, rooms or
availability zones.

The following table lists the number of runtime components in this
configuration.

|Assets                                 |Spec|
|----|----|
|Master nodes                           |3|
|Worker nodes                       |3|
|Cassandra nodes                        |3|
|RDBMS instance                         |2|
|Dedicated storage                  |70GB|
|Shared storage                     |20GB|
|External IP (public or private)    |1|
|Load balancer                          |1|
|External identity access management    |1|
|Docker registry                        |1|
|Bastion                                |1|
|Worker pipeline                        |1|
*Table 2: List of assets for high availability*

> These values are the minimum recommended starting point. Your actual values will depend on many factors, like the number of APIs, payload size, etc.

### VM sizes

These are the recommended parameters for the VMs:

-   Master node: 2vCPU and 8GB memory (for master node according to
Kubernetes sizing recommendation for 10 workers nodes; see complete
description [here](https://kubernetes.io/docs/setup/best-practices/cluster-large/))
-   Worker node (for API Gateway/Manager/ANM): 2vCPU and 8GB memory
-   Worker node (for monitoring tools): 2vCPU and 8GB memory
-   Cassandra node: 2vCPU and 8GB memory with 5GB/s
-   SQL database node (estimation based for Mysql): 2vCPU (mono-core) and 4GB memory

### Storage

SSD disks with low latency are recommended for storing data.

#### Cassandra and RDBMS

Cassandra is a distributed database, so each node requires its own
storage. Axway recommends a cluster with three nodes, so three volumes
of **50GB** must be allocated. RDBMS uses less storage so **20GB** for a
simple instance is acceptable.

#### Events logs

API gateways generate events. A shared volume is required. The volume is
limited by a setting inside the deployment package (FED file). By
default, this value is set to **1GB**. Three gateways are deployed in
the minimal configuration. But autoscaling can increase this number to
12 replicas. So, storage of **13 GB** must be configured.

Please, follow this method to estimate disk space:

![](/Images/apim-reference-architectures/container-aws/image4.png)
>Max_disk_space x API_gateway_max_replica + 1GB = recommended_disk_space

It is important to select a proper disk option for logs in your target
environment (cloud or on-premises). In the Kubernetes environment, there
may be many pods simultaneously writing to shared storage. Selected
disks must support this target workload.

#### Storage for logs

A volume is required to store all logs streamed out from a Kubernetes
cluster. The data will contain:

-   Kubernetes logs
-   Containers logs
-   Applications logs

>*FluentD* has been used in our environment to stream logs.

#### Total storage requirements

This table summarizes the total required storage space.

|Components         |Kind      | Size|
|------------------ |-----------| ---------------|
|Cassandra          |Dedicated   |50GB per node|
|SQL Database       |Dedicated   |20GB per node|
|Events             |Shared      |13GB|
|Storage for logs   |Dedicated   |100GB|
*Table 3: Details of disk spaces*

### Load balancer

A load balancer is required in front of the cluster. We use the
Kubernetes object called **ingress controller** that is responsible for
**fulfilling the ingress rules**. A layer 7 load balancer (AWS ALB) is
configured in this architecture. It performs important tasks of
**terminating TLS connection** and **request routing**. A layer 4 load
balancer is used inside a Kubernetes cluster to route all requests on a
specific IP.

![](/Images/apim-reference-architectures/container-aws/image5.JPG)

### Network

This typical network deployment is based on a minimal number of
segregated zones. It consists of four subnets:

-   Kubernetes master subnet
-   Bastion subnet where administration tasks will be done
-   Kubernetes worker nodes subnet
-   Data subnet to host all databases and other kinds of storage

![](/Images/apim-reference-architectures/container-aws/image6.png)
Each subnet must be protected by a firewall with
appropriate inbound and outbound rules. See details in the following
diagram (just one availability zone is shown) and table.

|ID|Description|From|To|Protocol|Ports|
|----|----|----|----|----|----|
|1a |This flow is the ingress between consumers and the load balancer	|Public network intranet |	Load balancer |TCP |443 |
|1b |Flow with ingress controller	 |Frontend subnet	 |K8S worker subnet |TCP |443 |
|2 |Connections from Axway components to Cassandra and API Analytics | K8S worker subnet|Data subnet |TCP |9042 3306 |
|3 |Usage of Kube-proxy to access the Kubernetes Dashboard |Bastion subnet |K8S master subnet |TCP |8001 |
|4 | Access to the administration web interface (ANM)|Bastion subnet |Frontal subnet |TCP |443 |
|5 |Access for administration database tasks |Bastion subnet |Data subnet |TCP |9042 3306 |
|6 |Access to bastion hosts |Intranet | Bastion subnet|TCP |3389 22 |
|7 | Egress flow to pull Docker images|K8S worker subnet |Public network intranet |TCP |443 |
| |Egress flow to send email to SMTP relay |K8S worker subnet |Public network intranet |TCP |465 587 |
| | Connection to external identity access management| K8S worker subnet|Public network intranet |TCP |443 |
|8 | Egress to pull Helm package on Docker registry|Bastion subnet |Public network intranet |TCP |443 |
|9 |Bidirectional communication with K8s API server on master nodes |K8S master subnet |K8S master subnet |TCP |443 |
*Table 4: Network flow rules*

Kubernetes considerations
-------------------------

This section focuses on additional Kubernetes objects and configuration
**inside the cluster** to support Axway components. This is a required
step before deploying containers.

### Deployment options

Some parameters are available only at the creation of the Kubernetes
cluster. The first is a network manager for communication between pods
and the second is a set of strong permissions for Kubernetes.

|Description|Type|
|----|----|
|Network CNI mode with a specific plugin (CALICO or Cloud provider) to secure pod connections with other applications or resources inside the cluster |Recommended|
|Secure Kubernetes with RBAC capabilities | Recommended|

#### Network plugin

By default in Kubernetes, there is no isolation between pods inside the
cluster. It's not a problem to deploy it on a dedicated cluster; default
Kubernetes networking will be enough. API management capabilities don't
require specific rules. But in some cases, API management may be
deployed with other back end or apps in the same cluster. In this case,
it's necessary to use a Container Networking Interface (CNI) plugin.

There are three kinds of policies that can be applied:

-   Drop communication by default

-   Allow connection to API gateway from specific namespaces

-   Allow connection to pods from specific monitoring tools

The Axway team used CALICO without any specific rule. It was simple to
deploy with a manifest.

#### RBAC Permission

RBAC permission is a secure mechanism to manage authorization inside
Kubernetes. It's recommended to set people or application permissions to
manage resources:

-   Allow Helm to manage resources
-   Allow worker nodes autoscaling
-   Allow specific users to view pods, to deploy pods, to access Kubernetes Dashboard
-   Allow cert-manager to pull and encrypt certificate
-   Allow Kubernetes to provide cloud resources, like storage or load balancer

This is a minimal configuration and you can define more specific
permissions with cluster roles or binding in the cluster.

### Namespaces

A namespace allows splitting of a Kubernetes cluster into separated
virtual zones. It's possible to configure multiple namespaces that will
be logically isolated from each other. Pods from different namespaces
can communicate with a full DNS pattern
*(\<service-name\>.\<namespace-name\>.svc.cluster.local*). A name is
unique within a namespace, but not across namespaces.

As mentioned in [Kubernetes
documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/),
a typical usage of namespaces is **separating projects and configured
objects deployed by different teams**.

Axway recommends deploying all API management components inside the same
namespace:

-   According to Kubernetes best-practice, the deployment is the
responsibility of API team

-   It's easiest to deploy the solution inside an existing cluster

-   You don't have to specify full DNS to call other components,
therefore, preventing errors. You just use a service name
(*\<service-name\>*).

|Description|Type|
|----|----|
|Provide all API management assets in the same namespace   |Recommended|

### Pod resource limits

Axway API Gateway runs in a **Java VM**. A Java VM pool is limited with
the Xmx parameter: If there is a memory leak, it can affect the entire
cluster. JVM heap size should be limited (see section 3.4 on page 21).

Kubernetes permits defining a CPU and memory limits for each pod to
protect the cluster. Setting up the limits is especially important in
case a cluster is shared with other apps. When scheduling a pod
deployment, Kubernetes uses these limits to ensure that resources for
the pod are available on a target node.

|Description|Type|
|----|----|
|Limit memory and CPU usage to protect the cluster                                  |Recommended|
|Change Xmx value and resources limitations according to the size of worker nodes   |Recommended|

These are the recommended initial limits for AMPLIFY API Management
components:
-   API Manager pod: 2cpu with 2GB memory and initial request of 0,5cpu with 0,5GB memory
-   Admin Node Manager: memory resource limit of 2GB memory

### Components healthcheck

Kubernetes provides a very useful feature called probes. There is one
probe to check if a **pod is ready to be used at startup** and another
to periodically check if a **container is still operational**. These
probes are respectively called "readiness probe" and "liveness probe."

These probes are configured on all ports (traffic and UI) and use the
HTTP/HTTPS protocol.

|Description|Type|
|----|----|
|Implement Kubernetes probes to manage container status in real time   |Required|

### Affinity and anti-affinity mode

Kubernetes pod allocation strategy is based on the nodes' resources
availability. Potentially, more than one pod replica can be deployed on
the same node. For an HA deployment, you want to spread your runtime
components across multiple nodes and availability zones. For this
reason, we recommend using a Kubernetes option called
**podAntiAffinity**. You should instruct Kubernetes [not]{.underline} to
schedule the same replicas on the same node if it is possible based on
the resource availability.

|Description|Type|
|----|----|
|Dispatch APIM pods across available nodes (monitoring node can be excluded from this rule)   |Required|

### Autoscaling

To properly increase or decrease the number of runtime components to
accommodate a workload, there are two scaling techniques used in the
reference architecture:

-   Nodes/VMs autoscaling
-   Kubernetes pod autoscaling

#### Node scaling

If the allocated number of nodes/VMs is not enough for increasing
traffic, there are different ways to scale them. We recommend using a
platform-provided mechanism to control this. On AWS we use **Auto
Scaling Groups**. There are many ways to design autoscaling. The Axway
team uses an approach where a new node is added when the Kubernetes
scheduler can't schedule a new pod on the existing nodes.

#### Horizontal Pod Autoscaler

Using Kubernetes Horizontal Pod Autoscaler (HPA) you can automatically
scale the number of API Gateway components. HPA uses a control loop that
checks selected utilization metrics every 15s (default value). There are
several options for triggering autoscaling. The average CPU utilization
can be used. It is set to a high enough value for optimal resource
usages. When CPU utilization exceeds this threshold, Kubernetes adds
more pods. You need to test what should be a good CPU utilization based
on your pod start-up time, traffic pattern and potential impact on the
overall performance. An average CPU utilization of 75 percent is a good
starting point. Keep in mind that to get HPA working, you need to define
resource limits (notice, that our CPU limit is 2cpu, see section 3.3.3
on page 18). This is an example of this setting in Helm:
```
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 75
```

### External traffic

We use the ingress controller to expose API management apps and services
to external clients. It dynamically creates externally reachable URLs
for desirable endpoints, load balance traffic between pods and terminate
TLS connections. This mode is available only for HTTP and HTTPS.

|Description|Type|
|----|----|
|Terminate TLS before a request reaches pods                                               |Required|
|Need specific DNS entry to route requests (solution with rewrite path isn't functional)   |Required|

A specific DNS entry is required to route requests to a service inside a
Kubernetes cluster. These are the exposed interfaces in our
configuration:

-   *anm.FQDN* for API Gateway Manager UI
-   *apimgmt.FQDN* for API Manager UI
-   *apimgr.FQDN* for API traffic

DNS records must match the certificate and ingress host configuration.
These records must target the external IP used for the Kubernetes entry
point.

{{% alert title="Important" color="warning" %}}
It’s not possible to use some rewrite-path like https://FQDN/Components/ to access the web interface
{{% /alert %}}

It's necessary to configure the following annotations for ingress
configuration:

-   Disable HTTP/2 if your ingress chooses it by default
-   Authorized entry port (recommended to allow only HTTPS)
-   An HTTPS redirection if HTTPS is authorized with the previous rules
-   Force usage of TLS protocol v1.2
-   Specify HTTPS protocol for a back end
-   Localization of each certificate
-   A range IP authorization (optional, but recommended only for private access)

Kubernetes lists available ingress controllers
[here](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).
For the AWS implementation, we used the AWS ALB Ingress Controller. But
you can pick any Ingress controller option that fits your requirements.

### Secrets

Without secrets, all passwords are set in clear in Manifest. Kubernetes
define "secret" objects to encode in base64 all sensitive information.
Using *secrets* is very useful for variables in containers, Docker
registry login and technical token for shared storage.

Here is a table to list all the secrets used by pods:

| |Admin Node Manager   |API Gateway Manager   |API Gateway Traffic   |Ingress controller|
|-|-------------------|--- -------------------- --------------------- --------------------- --------------------
|Public certificate      |                     |                     | |                      X|
|Docker registry login   |X                    |X                     |X||
|Cassandra user ID       |                     |X                     |X||
|Shared storage ID       |X                    |X                     |X||
|SGBDR user ID           |X||||
*Table 5: Kubernetes secrets list*

A more secure solution for handling sensitive data instead of secrets is
[*Sealed Secrets* for Kubernetes](https://github.com/bitnami-labs/sealed-secrets) artifacts.

Customers may also use a key management service available on their
platform of choice as a more secure solution to store sensitive data.

API Management implementation details
-------------------------------------

This chapter describes API management components in Kubernetes.
Following is a deployment diagram with a complete representation of the
API management objects:

![](/Images/apim-reference-architectures/container-aws/image7.png)


### Admin Node Manager

Admin Node Manager provides the API Gateway Manager web interface with
monitoring capabilities. This component requires an RDBMS to store
analytical data. It is accessed through an ingress controller. We
recommend that ANM is available from an internal and secure location
(i.e., through bastion node).

To build an ANM container, you need to provide the following:

-   HTTPS certificate. This certificate will be used inside the cluster,
so it's possible to use a self-signed certificate. However, we
recommend reviewing the usage of self-signed certs with your
security team.

-   A license file with the EMT mode. Notice it is not required to start
Admin Node Manager, but may be required for optional features such
as FIPS mode.

-   JDBC library for the selected RDBMS.

-   RDBMS connection parameters.

To deploy you need to provide:

-   Heap size memory as mentioned before (set the value at 1024)

-   Persistent storage with read/write multiple pods capabilities to store events

-   RDBMS parameters

-   Log format in JSON

-   Trace level setting

The Helm charts contain a precheck mechanism (*initContainers*) to check
for required preconditions before this pod can be started. Those checked
preconditions are:

-   Up and running RDBMS server

Pod characteristics:

-   Healthcheck: LivenessProbe and ReadinessProbe

-   Kubernetes object: deployment

-   Resources limit: 1cpu & 2GB memory

-   Java heap size: 1024 MB

-   Automatic scalability: no

-   Replicas: 1

### API Manager UI

This pod supports an API Manager web interface (port 8075).

{{% alert title="Important" color="warning" %}}
As of Axway API Management v7.7, we recommend running only one API Manager UI pod. This pod is not used to process client requests, so using one pod is enough. It also simplifies architecture.
{{% /alert %}}

To build an API Manager container, you need to provide:

-   HTTPS certificate. This certificate will be used inside the cluster. It's possible to use a self-signed certificate. However, we
recommend reviewing the usage of self-signed certs with your security team.

-   A license file with the EMT mode set.

-   A group name. It's mandatory because Admin Node Manager needs it to manage gateway.

-   JDBC library for the selected RDBMS.

-   A FED file with server settings and policies. It is also possible to use policy and environment package files. So, you can have one common policy package and several environment packages: one for each deployment environment.

To deploy you need to provide:

-   Cassandra hosts and Keyspace name

-   Heap size memory as mentioned before (set the value at 1024)

-   ANM hostname

-   RDBMS parameters

-   Log format in JSON

The Helm charts contain a precheck mechanism (*initContainers*) to check
for required preconditions before this pod can be started. Those checked
preconditions are:

-   Up and running three Cassandra hosts

-   Up and running RDBMS server

-   Up and running ANM pod

Pod characteristics:

-   Healthcheck: LivenessProbe and ReadinessProbe

-   Kind: deployment

-   Exposed port UI 8075 (customers may configure a different port if needed)

-   Resources limits: 2cpu & 2GB memory

-   Java heap size: 1024 MB

-   Session affinity: by cookie

-   Replicas: 1

API Manager UI pod can send an email via SMTP relay. It's configured
inside a FED file.

### API Gateway/Manager (traffic)

API Gateway/Manager pod handles API calls. At the least, a traffic port
(port 8065) should be exposed to this container. But if needed,
additional ports can be exposed (secure and nonsecure ports 8443, 8080
and 8081). This component uses a volume mount point with read/write
multiple pods capabilities to store events. Other data to persist is
streamed out by *FluentD*. With this approach, you can reduce the size
of persistence data required for log/even data.

The API Gateway/Manager container is the same as the API Manager UI
container. The deployment parameters are the same. The only differences
are in the Helm charts configuration: these are primarily the number of
replicas and exposed ports.

{{% alert title="Important" color="warning" %}}
During development of the policies and server settings, you should use EMT_DEPLOYMENT_ENABLED flag when a container is started. This flag enables direct deployment of the Gateway configurations from Policy Studio. You will be able to develop and test your policies as with the classic deployment option. This option is recommended for running a small test environment using ***only one*** API gateway pod on a developer’s machine.
It’s important ***not to use*** this flag in a production environment.
Any testing in an upper environment (Test, QA, etc.) should be done ***using a Docker image*** build for that environment.
{{% /alert %}}

The Helm charts contain a precheck mechanism (*initContainers*) to check
for required preconditions before this pod can be started. Those checked
preconditions are:

-   Up and running three Cassandra hosts

-   Up and running RDBMS server

-   Up and running ANM pod

-   UP and running API Manager UI pod

Pod characteristics:

-   Healthcheck: LivenessProbe and ReadinessProbe

-   Kind: deployment

-   Exposed traffic port 8065 (customers may configure a different port if needed)

-   Resources limits: 2cpu & 2GB memory

-   Java heap size: 1512 MB

-   Automatic scalability: yes

-   Replica number: minimum/initial value is 3

Cassandra considerations
------------------------

Cassandra must be hosted outside the cluster. Minimum three nodes are
required: one node in each availability zone. Please see [the official
documentation](https://docs.axway.com/bundle/APIGateway_77_CassandraGuide_allOS_en_HTML5/page/Content/CassandraTopics/CassandraStartPage.htm) on the Axway support website.

Security considerations
-----------------------

The following is a summary of security considerations discussed within
this document:

-   In layered infrastructure, the network is protected by firewall LVL4
that filters only specifics ports. Any administrative access to
infrastructure components is enabled through a subnet called a
bastion.

-   All components are deployed in a cluster mode. Kubernetes uses RBAC
permission to set roles per user or group. CNI plugin protects
communications inside the cluster. All external connections are
protected by an ingress controller with a public certificate.

-   In the application layer, sensitive data is protected inside a
secret. All communications are encrypted in a cluster. API
Management uses only TLS 1.2.

-   Do not run containers as privileged.

-   Only allow communication on required ports.

SQL database considerations
---------------------------

AMPLIFY API Management supports several RDBMS:

-   MySQL or MariaDB

-   Microsoft SQL Server

-   Oracle database

For AWS deployment, API Manager uses Amazon RDS for MySQL to store
analytics data. It is configured with High Availability on two nodes
across two different availability zones. A MySQL schema is packaged
along with the Helm charts. It is imported by running a Kubernetes job
that runs a *dbsetup* command to create a schema. It is also possible to
import a schema through a MySQL client.

Logging/tracing
---------------

The following logs should be persisted:

-   Trace files

-   Admin Node Manager: `INSTALL\_DIR/trace`

-   API Gateway instance: `INSTALL\_DIR/groups/\<group-id\>/\<instance-id\>/trace`

-   Transaction audit log -- see [this documentation](https://docs.axway.com/bundle/APIGateway_77_AdministratorGuide_allOS_en_HTML5/page/Content/AdminGuideTopics/log_global_settings.htm).

-   Transaction access log -- see [this documentation](https://docs.axway.com/bundle/APIGateway_77_AdministratorGuide_allOS_en_HTML5/page/Content/AdminGuideTopics/log_access_settings.htm).

-   Transaction event log -- see [this documentation](https://docs.axway.com/bundle/APIGateway_77_AdministratorGuide_allOS_en_HTML5/page/Content/AdminGuideTopics/log_event_settings.htm)

-   Open traffic event log -- see [this documentation](https://docs.axway.com/bundle/APIGateway_77_AdministratorGuide_allOS_en_HTML5/page/Content/AdminGuideTopics/log_open_traffic_event_settings.htm)

Environmentalization and Promotion
----------------------------------

Environmentalization and promotion go hand in hand. AMPLIFY API
Management uses two different artifacts:

1.  Polices -- used by API Gateway

2.  API data files -- used by API Manager

![](/Images/apim-reference-architectures/container-aws/image8.png)
The section below covers how the polices
are promoted from the development environment to the testing
environment.

Policy promotion steps:

1.  Policy developer edits configuration and deploys in development
environment via Policy Studio.

2.  Test the polices in development environment.

3.  Enable the display of configuration settings that are assigned for
environmentalization in Policy Studio.

4.  Open *Window \> Preferences \> Environmentalization* in the main
menu, and select *Allow environmentalization* *of fields*

5.  Policy developers environmentalize environment-specific settings.
For example:

-   *URL*, *User Name*, and *Password* fields in a Default Database Connection
-   *URL* field in a Connect to URL filter
-   *X.509 Certificate* field in an HTTPS interface
-   *URL*, *User Name*, *Password*, and *Signing Key* fields for an LDAP configuration

6.  Check-in the code to git.

7.  CI pipeline checks out the code from git and uses [projpack](https://docs.axway.com/bundle/APIGateway_77_PromotionGuide_allOS_en_HTML5/page/Content/DeployPromoGuideTopics/deploy_package_tools.htm)
CLI to create a pol and env package. This can be done in the Policy
Studio. When the active configuration is loaded, `Select File > Save > Policy Package` to save the pol file and `Select File > Save > Environment Package` to save the env file.

8.  Policy Developer uses env package to create an environment-specific
artifact (like test.env and prod.env).

9.  Create a Docker container using pol and env package. See [this documentation](https://docs.axway.com/bundle/APIGateway_77_ContainerGuide_allOS_en_HTML5/page/Content/ContainerTopics/docker_script_gwimage.htm)
for more info.

Example

```
./build_gw_image.py --license=license.lic --default-cert
  --pol=banking.fed \
  --env=test.env --merge-dir /home/axway/apigateway
```

10. Push Docker image to your Docker registry.

11. Deploy to test environment by pulling an environment-related tag
from the Docker registry.

API Manager promotion:

-   API as code. The GitHub project can be used to create and promote an
API across environments.
>https://github.com/Axway-API-Management-Plus/apim-cli.

-   Promote API using export and import mechanism - [refer to this
document](https://docs.axway.com/bundle/APIManager_77_APIMgmtGuide_allOS_en_HTML5/page/Content/APIManagementGuideTopics/api_mgmt_promote.htm).

-   Promote API using REST API. Sample implementation -
<https://github.com/Axway-API-Management-Plus/apim-deployment>.

Infrastructure components for AWS deployment
--------------------------------------------

These are the infrastructure components that the Axway team has tested
and recommends for the initial configuration of the reference
architecture:

-   Kubernetes master and worker nodes - *m5.large* EC2 instances

-   Cassandra nodes -- *m5.large* EC2 instances

-   RDBMS -- Amazon RDS running on *db.t3.large* EC2 instances

Performance testing
-------------------

The Axway team ran a variety of performance tests on the reference
architecture. These tests were executed with testing tools Postman or
JMeter. A test is configured for **300 seconds** with a **loop of
5 000 000 repetitions** and **20 threads**.

|Message size   |Configuration   |Minimal Threshold (transactions/sec)   |Results (transactions/sec)|
|-------------- |--------------- |-------------------------------------- |----------------------------|
|1Kb            |Passthrough     |>1270                                 |1270 <> 1930|
|               |API key         |>800                                  |800 <> 1465|
|               |OAuth           |>745                                  |1075 <> 1337|
|10kb           |Passthrough     |>1200                                 |1340 <> 1600|
|               |API key         |>750                                  |781 <> 1535|
|               |OAuth           |>670                                  |670 <> 724|
|50kb           |API key         |>300                                  |~750|
|100kb          |Passthrough     |>100                                  |150|
*Table 6: Performance validation threshold*

Maintenance
===========

After you deploy AMPLIFY API Management in production, you will need to
update product configurations (policies and settings) and install fixes
and service packs. This section outlines best practices in maintaining
your installation.

New configurations
------------------

With a shift to a container-based deployment, the notion of pushing API
Gateway/Manager configuration updates directly to the running instances
has changed. Now you need to create a new Docker image that contains the
latest Gateway/Manager configuration. Using [Kubernetes rolling
upgrades](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/),
you deploy a new Docker image to a cluster without interrupting your
request processing. The following guide explains how to create and
deploy a new Docker image:

<https://docs.axway.com/bundle/APIGateway_77_ContainerGuide_allOS_en_HTML5/page/Content/ContainerTopics/container_development.htm>

In general, the process of building AMPLIFY API Management Docker images
for installation can be depicted as in the picture below.

![](/Images/apim-reference-architectures/container-aws/image9.tmp)

Description of the images:

-   A base image includes a base product installation and doesn't change
frequently. You update a base image only when you need to install an
update for the underlying operating system, or when you need to
install a service pack for AMPLIFY API Management or upgrade the
product.

-   An ANM image represents Admin Node Manager, but in the
container-based deployment, it is not used for pushing new
configurations to the gateways. It is primarily a monitoring tool.

-   Gateway or API Manager images contain configuration and settings
(*.pol + .evn* , or *.fed*) that are updated frequently. These are
the type of images that you would rebuild frequently.

Axway provides sample build scripts as a working example to be modified
and used by customers. The sample scripts are provided from the Axway
support site. For example, a download file for AMPLIFY API Management
v7.7 is:

`APIGateway\_7.7-1\_DockerScripts.tar.gz`

To streamline this process for building Docker images, Axway recommends
creating a CI/CD pipeline that should, at least, include these tasks:

-   Creating a policy package (.pol)

-   Creating an environment package (.env) for every target deployment
environment

-   Creating an environment-specific Docker image; for example, for a
test, QA, or production

-   Deploying a new Docker image to a target environment and smoke
testing

Customers should use a source control management (SCM) system for
maintaining/versioning policy and environment packages and a Docker
registry for Docker images.

Product updates
---------------

There are variations in how to install a new patch, service pack, or
upgrade for a new AMPLIFY API Management version. But the overall
approach is that:

-   For an upgrade or SP installation, you rebuild your base, ANM and
Gateway/Manager images

-   For a patch installation, you rebuild your ANM and Gateway/Manager
images

### Installing a patch

Patch installation requires updating one or more files in the target
product installation directory. To apply a patch, you need to overwrite
one or more files when you build an ANM or Gateway/Manager images. The
build scripts provide the `--merge-dir` option that allows you to
overwrite any file(s) in a Docker image. The example and additional
explanation of the procedure can be found in [this section](https://docs.axway.com/bundle/APIGateway_77_ContainerGuide_allOS_en_HTML5/page/Content/ContainerTopics/container_patch_sp.htm).

The following is an example of a build command for a Gateway/Manager
image:

```
./build\_gw\_image.py
  --license=/tmp/api_gw.lic
  --domain-cert=certs/mydomain/mydomain-cert.pem
  --domain-key=certs/mydomain/mydomain-key.pem
  --domain-key-pass-file=/tmp/pass.txt
  --parent-image=my-base:latest
  --fed=my-group-fed.fed
  --fed-pass-file=/tmp/my-group-fedpass.txt
  --group-id=my-group
  ***--merge-dir=/tmp/apigateway***
```

![](/Images/apim-reference-architectures/container-aws/image10.png)

The *--merge-dir* points to a directory that contains
file(s) that will replace specific installation files on a target Docker
image. A merge directory must be named ***apigateway***. For example, if
you need to update the *envSettings.props* file in a Gateway image, this
is what you should have in your merge directory:

The */tmp/apigateway* directory should mirror the file path in your
target Docker image.

#### Step-by-step example

Let's look at applying one of the patches for APIM v7.7 -- *APIGateway7.7-SP1 Patch17276*:

1.  Download a patch from the support website

2.  Create a merge directory:

`mkdir /tmp/apigateway`

3.  Extract downloaded file into the merge directory:

```
tar -xvzf APIGateway\_7.7-SP1\_Patch17276\_d16e79fb\_allOS\_BN20191024.tgz -C /tmp/apigateway/
```

4.  If you look at the merge directory, you will see that two files will be written to a target Docker image during build:
![](/Images/apim-reference-architectures/container-aws/image11.png)

5.  Now you can run your build script specifying the merge directory
that you've created:

```
./build\_gw\_image.py
  --license=/tmp/api\_gw.lic
  --domain-cert=certs/mydomain/mydomain-cert.pem
  --domain-key=certs/mydomain/mydomain-key.pem
  --domain-key-pass-file=/tmp/pass.txt
  --parent-image=my-base:latest
  --fed=my-group-fed.fed
  --fed-pass-file=/tmp/my-group-fedpass.txt
  --group-id=my-group
  ***--merge-dir=/tmp/apigateway***
  --out-image=my-gtw:7.7-SP1-p17276
```

6.  Two files from the patch will overwrite (or create) corresponding
files in the new Docker image.

### Installing a service pack

Installing a service pack is identical to creating your first API
Gateway/Manager Docker image. The only difference is that you just need
to download and use a combined installation file that includes base
product plus a corresponding service pack. As an example, we look at
AMPLIFY API Management v7.7 and API Management v7.7 with SP1install
files:

-   API Management v7.7 install is titled: *API Gateway and API Manager 7.7 Install (linux-x86-64)* with the following file - `APIGateway\_7.7\_Install\_linux-x86-64\_BN4.run`

-   API Management v7.7 with SP1install is titled: *API Gateway 7.7 Install Service Pack 1 (linux-x86-64)* with the following file - `APIGateway\_7.7\_SP1\_linux-x86-64\_BN201908271.run`

The build process will be identical to the one described in section 4.1
New configurations.

### Upgrading the product

Upgrading your existing deployment to a new version of AMPLIFY API
Management will be similar to a process described in section *4.1* *New
configurations*. But there are some additional steps to migrate your
existing Gateway/Manager configuration using Policy Studio. The
[following section](https://docs.axway.com/bundle/APIGateway_77_ContainerGuide_allOS_en_HTML5/page/Content/ContainerTopics/container_upgrade.htm)
in the documentation describes this process for API Management v7.7. The
extra steps are needed to import your existing .fed file in Policy
Studio. This will trigger an automatic update to your FED file. When
done, export the updated FED (or *.pol* and *.env*) file and use it for
building a new Docker image(s).

Notice, that there may be additional upgrade steps related to other
parts of your deployment (e.g. Cassandra or RDBMS).

### Adding customization

You may want to add or customize some files in the product installation
directory. To do this with a container-based deployment, follow steps
outlined in the section 4.2.1 Installing a patch. The only difference is
the need to create manually a proper directory structure of your merge
directory.

Pushing a new Docker image to your Kubernetes cluster
-----------------------------------------------------

When there is time to push a new image to a Kubernetes cluster (in any
environment), we rely on Kubernetes rolling updates that are supported
under the **Deployment** object. This object has a section that governs
the process of updating running containers with a new Docker image. This
is an example of an update strategy specification:
```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
```

With the rolling update, Kubernetes will take down a designated number
of pods, update them with a new image, and start. Then continue this
process with other pods. The optional *maxUnavailable* parameter
specifies that only one pod at a time can be updated. Instead of an
absolute value for *maxUnavailable,* you can provide a percentage value
that will represent a portion of your Deployment pods that can be
updated at once. With four pods in Deployment set and *maxUnavailable:
25%* (the default value for *maxUnavailable* if it is omitted),
Kubernetes will update one pod at a time.

There is another optional parameter -- *maxSurge* (default value is 25
percent). For example, if there are four pods in your Deployment, then
during a rolling update, the maximum number of pods with old and new
configurations combined can't be more than 5:

`4 pods (100%) + 1 pod (25% - maxSurge) = 5 pods (125%)`

You can read the [following articles](https://kubernetes.io/blog/2018/04/30/zero-downtime-deployment-kubernetes-jenkins/)
to better understand rolling updates and other techniques for a zero
downtime deployment.

Configuration and data backups
==============================

It is essential for the smooth operation of an API management solution
to perform regular backups of data and configurations. At a minimum, the
following data, source code and artifacts should be included:

-   Policies and server configuration -- there are a couple of ways to
maintain your deployment artifacts. You can maintain an API Gateway
configuration as a Policy Studio folder in a source code management
system (SCM). GitHub and GitLab are two popular choices. As an
alternative, you can version control FED or POL/ENV files.

In addition to the source code, you need to maintain deployment
artifacts in the form of a Docker image, since this is what you deploy
in the EMT mode. Thus, a Docker registry is required.

-   RDBMS backup - use the appropriate backup procedure for the selected
RDBMS.

-   Cassandra -- follow recommendations in the official documentation
for the Cassandra backup (see [this](https://docs.axway.com/bundle/APIGateway_77_CassandraGuide_allOS_en_HTML5/page/Content/CassandraTopics/cassandra_BUR.html)).
You will need to decide how frequently you should back up Cassandra.
This decision will be impacted by what data you store in Cassandra.
Is it only API Manager data? Or does it include OAuth tokens and
custom KPS? The main discussion point is to decide how much data you
could potentially lose without seriously affecting your business. A
daily backup may be a good starting point.

There are a couple of good blogs on backing up and restoring Cassandra:

-   [Cassandra Backup and Restore - Backup in AWS using EBS Volumes](https://thelastpickle.com/blog/2018/04/03/cassandra-backup-and-restore-aws-ebs.html)

-   [Medusa - Spotify's Apache Cassandra backup tool is now open source](https://thelastpickle.com/blog/2019/11/05/cassandra-medusa-backup-tool-is-open-source.html)

-   Log/trace/even files -- your backup strategy will depend on your
choice of infrastructure. For running on AWS, we use AWS S3 service
that has high durability and reliability metrics (see [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/DataDurability.html)).
Log files are not required to run API Gateway. Thus, if you use AWS
S3 service (or similar), there is no need to back up log files. If
log file storage is temporarily unavailable, it shouldn't affect the
runtime.

-   Helm charts and other environment-specific files should be part of
an SCM with its own backup mechanism.

Disaster recovery
=================

For a disaster recovery procedure, you should have access to cloud
resources in another region. Using backed-up configurations/data and
Docker registry, you should be able to run a CI/CD pipeline for creating
a new Kubernetes cluster in another region.

Known constraints and roadmap
=============================

As of AMPLIFY API Management v7.7, there are some differences or
constraints compared to the classic mode deployment:

-   API Portal and Embedded Analytics are not yet supported in the EMT
mode

-   Distributed Ehcache is not supported. However, you can use Apache
Cassandra as a distributed data store where CRUD operations are
supported to directly interact with KPS, using scripts.

-   Running with Embedded ActiveMQ configured in an instance is not
advised. The recommendation is to use an "external" JMS provider.

The current version does not support all the configurations that
currently exist for the classic deployment. Axway plans to address these
limitations by delivering the following capabilities in the future:

-   Broader support of current APIM configurations

-   Running API Portal in a Docker container

-   Native support of Embedded Analytics

-   Advancements in the current EMT mode

-   Mutable policy configuration - that enables a policy to be
modified without the need to bake in a new Docker image

-   Reference architecture document for other cloud providers

You can view all roadmaps on Axway Community Portal
[here](https://community.axway.com/s/product-roadmaps).

Appendix A -- Glossary of Terms
===============================

|Reference   |Description|
|----------- |----------------------------------------------------------------------------------------------|
|SAML        |Security Assertion Markup Language -- based on XML, this protocol permits SSO authentication|
|SSO         |Single Sign-On -- unique authentication for a user|
|K8S         |Short name of Kubernetes product|
|FED         |The file extension of the federated deployment package (policy + environment packages)|
|POL         |The file extension of a policy package file|
|ENV         |File extension of an environment package file|
