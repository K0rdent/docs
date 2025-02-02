# k0rdent architecture

The k0rdent architecture follows a declarative approach to cluster management using the principles of Kubernetes Cluster API (CAPI), where you define the state you want using YAML and CAPI ensures you get it. For example, if you wanted to create a new cluster, you’d define that cluster just as you’d define a Pod or other Kubernetes object, then apply the YAML file to the Management Cluster, which acts as the heart of the system. 
The Management Cluster can orchestrate the provisioning and lifecycle of multiple child clusters on multiple clouds and infrastructures, keeping you from having to directly interact with individual infrastructure providers. By abstracting infrastructure, k0rdent promotes reusability (reducing, for example, the effort required to implement an IDP on a particular cloud), encourages standardization where practical, lets you use the clouds and technologies you want, but also minimizes the cost of switching (e.g., open source subsystems, cloud substrates, etc.) if you need to.

The k0rdent architecture comprises the following components:

* **Cluster Management:** Tools and controllers for defining, provisioning, and managing clusters.
* **State Management:** Controllers and systems for monitoring, updating, and managing the state of child clusters and their workloads.
* **Infrastructure Providers:** Services and APIs responsible for provisioning resources such as virtual machines, networking, and storage for clusters.
* **Templates:** Templates that can be used to define and create managed child clusters or the workloads that run on them.

Let’s look at each of these in more detail.

![k0rdent Architecture - Simplified](assets/k0rdent-architecture-simple.png)


## Management cluster

The management cluster is the core of the k0rdent architecture. It hosts all of the controllers needed to make k0rdent’s platform engineering work. This includes:

* **k0rdent Cluster Manager (KCM) Controller:**  KCM provides a wrapper for k0rdent’s CAPI-related capabilities. It orchestrates:
    * **Cluster API (CAPI) Controllers:** CAPI controllers are designed to work with specific infrastructure providers. For example, one CAPI controller will manage the creation and lifecycle of Kubernetes clusters running on Amazon Web Services, while another manages those on Azure. It’s also possible to create custom CAPI controllers to integrate with internal systems.
    * **k0smotron Controller:** k0smotron extends CAPI with additional functionality, including control plane and worker node bootstrap providers for k0s Kubernetes, and a control plane provider that supports Hosted Control Plane creation (i.e., k0s control planes in pods on a host Kubernetes cluster, which can be the same cluster that hosts k0rdent).The k0smotron project has also provided a so-called ‘RemoteMachine’ infrastructure provider for CAPI, enabling deployment and cluster operations via SSH on arbitrary remote Linux servers (including small-scale edge devices).
    clusters. For example, users can deploy ingress controllers or other tools, or they can deploy complete application workloads.
* **k0rdent Service Manager (KSM) Controller:** KSM is responsible for lifecycle managing (deploy, scale, update, upgrade, teardown) services and applications on clusters, and for doing continuous state management of these services and applications. It orchestrates:
    * **Addon Controller:** Responsible for coordinating addons (combinations of services (and infrastructure provisioning dependencies) that add capabilities to the platform, e.g., KubeVirt with its dependencies can be packaged as an addon &mdash; installing it makes the cluster's worker nodes able to host virtual machines as containers). Artifacts for addons are stored locally or remotely in a Helm repo and OCI repository.
    * **Helm Controller:** Responsible for directly managing Helm operations involved in lifecycle managing services and applications on clusters.
    * **Event Controller:** Responsible for polling the state of services and passing notifications to a Notification Provider.
* **k0rdent Observability & FinOps (KOF) Controller (not depicted in above diagram):** k0rdent Observability and FinOps provides enterprise-grade observability and FinOps capabilities for k0rdent-managed Kubernetes clusters. It enables centralized metrics, logging, and cost management
through a unified OpenTelemetry-based architecture.

We’ll take a closer look at these pieces under [Roles and Responsibilities](#roles-and-responsibilities).

## Cluster Deployments

A cluster deployment is also known as a child cluster, or a workload cluster. It’s a Kubernetes cluster provisioned and managed by the management cluster, and it’s where developers run their applications and workloads. These are “regular” Kubernetes clusters, and don’t host any management components. Clusters are fully isolated via namespaces from the management cluster, and also from each other, making it possible to create multi-tenant environments. 

You can tailor a child cluster to specific use cases, with customized addons such as ingress controllers, monitoring tools, and logging solutions. You can also define specific Kubernetes configurations (for example, network policies, storage classes, and security policies) so they work for you and your applications or environments.

Simply put, child clusters are where applications and workloads run.

## Templates

One of the important tenets of the platform engineering philosophy is the use of Infrastructure as Code, but k0rdent takes that one step further through the use of templates. Templates are re-usable text definitions of components that can be used to create and manage clusters. Templates provide a declarative way for users and developers to deploy and manage complex clusters or components while massively reducing the number of parameters they need to configure. Considered generally, k0rdent templates are:

* **Formatted using YAML:** Templates use YAML as an abstraction to represent the target state, so they’re human-readable and editable.
Designed to be used in multiple contexts using runtime parameterization: Through the use of placeholders, you can customize the templates at runtime without having to directly edit the template.
* **Used for both cluster creation and addon management:** Users can define a cluster using YAML, or they can define addons, such as an ingress operator or monitoring tools, to be added to those clusters.
* **Of limited scope:** k0rdent lets you set restrictions over what templates can be deployed by whom. For example, as the platform manager (see Roles and Responsibilities), you can specify that non-admin users can only execute templates that deploy a particular set of controllers.

Major template types used in k0rdent include:

* **ClusterTemplates:** ClusterTemplates define clusters in coordination with the clouds and infrastructures they run on. They're designed to be immutable &mdash; they get invoked by k0rdent objects like `ClusterDeployment`s to create and manage individual clusters and groups of clusters.
* **ServiceTemplates:** ServiceTemplates define services, addons, and workloads that run on clusters. They're also designed to be immutable, and get invoked by 'ClusterDeployment's and other k0rdent objects so that IDPs/platforms can be declared and managed as units. 

## Roles and responsibilities

k0rdent was designed to be used by several groups of people, with hierarchical and complementary roles and responsibilities. You may have your own names for them, but we’ll refer to them as:

* **Platform Architect:** This person or team has global responsibility to the business and technical stakeholders for designing IDPs/platforms for later adaptation to particular clouds and infrastructures, workloads, performance and cost objectives, security and regulatory regimes, and operational requirements. k0rdent enables Platform Architects to create sets of reusable ClusterTemplates and ServiceTemplates closely defining IDPs/platforms in the abstract.
* **Platform Lead:** This person or team (sometimes referred to as 'CloudOps') is primarily responsible for the left-hand side of the above diagram, corresponding to k0rdent Cluster Manager (KSM). They adapt ClusterTemplates to the correct cloud, and they make sure that everything is working properly. They’re also responsible for limiting the Project Team’s access to Cluster and Service templates necessary to do their jobs. For example, they might limit the templates that can be deployed to an approved set, or provide CAPI operators for only the clouds on which the company wants applications to run, helping to eliminate shadow IT. 
* **Project Team:** This person or team is responsible for the right-hand side of the above diagram, corresponding to k0rdent Service Manager. They use ClusterTemplates and ServiceTemplates provided by the Platform Lead (as authorized to do so) and may create additional ServiceTemplates to customize their own Kubernetes cluster so that it’s appropriate for their application.

## Credentials

Creating and managing Kubernetes clusters requires having the proper permissions on the target infrastructure, but you certainly wouldn’t want to give out your AWS account information to every single one of your developers.

To solve this problem, k0rdent lets you create a Credential object that provides the access your developers need. It works like this:
The platform lead creates a provider-specific ClusterIdentity and Secret that include all of the information necessary to perform various actions.
The platform lead then creates a Credential object that references the ClusterIdentity.

Developers can reference the Credential object, which gives the cluster the ability to access these credentials (little “c”) without having to expose them to developers directly.

## TL;DR - conclusion

k0rdent provides a comprehensive Kubernetes lifecycle management framework through its three core components:

* **KCM:** Cluster provisioning and management.
* **KSM:** Application and runtime state management.
* **KOF:** Observability, logging, and cost optimization.

With multi-provider support, templated deployments, and strong security controls, k0rdent is being built to enable scalable, efficient, and consistent Kubernetes operations.