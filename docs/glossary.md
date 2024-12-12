# Glossary

This glossary is a collection of terms related to Project 2A. It clarifies some
of the unique terms and concepts we use or explains more common ones that may
need a little clarity in the way we use them.

### Beach-head services
We use the term to refer to those Kubernetes services that need to be installed
on a Kubernetes cluster to make it actually useful, for example: an ingress
controller, CNI, and/or CSI. While from the perspective of how they are deployed
they are no different from other Kubernetes services, we define them as distinct
from the apps and services deployed as part of the applications.

### Cluster API (CAPI)
CAPI is a Kubernetes project that provides a declarative way to manage the 
lifecycle of Kubernetes clusters. It abstracts the underlying infrastructure, 
allowing users to create, scale, upgrade, and delete clusters using a 
consistent API. CAPI is extensible via providers that offer infrastructure- 
specific functionality, such as AWS, Azure, and vSphere.

### CAPI provider (see also [Infrastructure provider](#infrastructure-provider-see-also-capi-provider))
A CAPI provider is a Kubernetes CAPI extension that allows 2A to manage and
drive the creation of clusters on a specific infrastructure via API calls.

### CAPA
CAPA stands for Cluster API Provider for AWS.

### CAPV
CAPV stands for Cluster API Provider for vSphere.

### CAPZ
CAPZ stands for Cluster API Provider for Azure.

### Cloud Controller Manager
Cloud Controller Manager (CCM) is a Kubernetes component that embeds logic to
manage a specific infrastructure provider.

### ClusterIdentity
ClusterIdentity is a Kubernetes object that references a Secret object
containing credentials for a specific infrastructure provider.

### Credential
A `Credential` is a custom resource (CR) in HMC that supplies 2A with the
necessary credentials to manage a specific infrastructure. The credential object
references other CRs with infrastructure-specific credentials such as access
keys, passwords, certificates, etc. This means that a credential is specific to
the CAPI provider that uses it.

### Hosted Control Plane (HCP)
An HCP is a Kubernetes control plane that runs outside of the clusters it
manages. Instead of running the control plane components (like the API server,
controller manager, and etcd) within the same cluster as the worker nodes, the
control plane is hosted on a separate, often centralized, infrastructure. This
approach can provide benefits such as easier management, improved security, and
better resource utilization, as the control plane can be scaled independently
of the worker nodes.

### Infrastructure provider (see also [CAPI provider](#capi-provider-see-also-infrastructure-provider))
An infrastructure provider (aka `InfrastructureProvider`) is a Kubernetes custom
resource (CR) that defines the infrastructure-specific configuration needed for
managing Kubernetes clusters. It enables Cluster API (CAPI) to provision and
manage clusters on a specific infrastructure platform (e.g., AWS, Azure, VMware,
OpenStack, etc.).

### Managed cluster
A Kubernetes cluster created and managed by Project 2A.

### Mutli Cluster Service
The `MultiClusterService` is a custom resource used to manage deployment of beach-head services
across multiple clusters.

### Management cluster
The Kubernetes cluster where 2A is installed and from which all other managed
clusters are managed.
