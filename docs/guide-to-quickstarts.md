# Guide to QuickStarts

The following QuickStart chapters provide a recipe for quickly installing and trying k0rdent. Setting up k0rdent for production is detailed in the [Administrator Guide](admin-before.md).

## What the QuickStart covers

The goal of the QuickStart is:

* To get a working environment set up for managing k0rdent.
* To get a Kubernetes cluster and other tools set up for hosting k0rdent itself.
* To select a cloud environment (AWS or Azure) and configure k0rdent to lifecycle manage clusters on this substrate.
* To use k0rdent to deploy a managed cluster.
* (Optional stretch goal) It's also possible to set up the k0rdent management cluster to lifecycle manage clusters on _both_ cloud environments.

## Where the QuickStart leads

The QuickStart shows and briefly explains the hows, whys, and wherefores of manually setting up k0rdent for use. Once built and validated, the QuickStart setup can be leveraged to begin [an expanding sequence of demos](https://github.com/k0rdent/demos) that let you explore k0rdent's many features. The demos presently use makefiles to speed and simplify setup and operations. We strongly recommend exploring this fast-evolving resource.

## QuickStart Prerequisites

QuickStart prerequisites are simple &mdash; you'll need:

* A desktop or virtual machine running a [supported version of linux](#supported-operating-systems) &mdash; This machine will be used to install a basic Kubernetes working environment, and to host a single-node k0s Kubernetes management cluster to host k0rdent components. For simplest setup, configure this machine as follows:
  * A minimum of 32GB RAM, 8 vCPUs, 100GB SSD (e.g., AWS `t3.2xlarge` or equivalent)
  * Set up for SSH access using keys (standard for cloud VMs)
  * Set up for passwordless sudo (i.e., edit /etc/sudoers to configure your user to issue sudo commands without a password challenge)
  * Inbound traffic - SSH (port 22) and ping from your laptop's IP address
  * Outbound traffic - All to any IP address
  * Apply all recent updates and upgrade local applications (sudo apt update/sudo apt upgrade)
  * (Optional) snapshot the machine in its virgin state
* Administrative-level access to an AWS or Azure cloud account - Depending on which cloud environment you prefer. k0rdent will leverage this cloud to provide infrastructure for hosting managed clusters.

### Supported Operating Systems

Any linux based os that supports deploying [k0s](https://k0sproject.io/) will work, you may need to adjust the suggested commands.


| OS | Package Manager | Link|
|----|-----------------|-----|
|Ubuntu Server| `apt` | [22.04.5 LTS, Jammy Jellyfish](https://releases.ubuntu.com/jammy/) |

> NOTE: Other recent versions of 'enterprise' Linux should work with the
> following instructions as well, though you will need to adapt for
> different package managers and perhaps use slightly-different
> provider-recommended methods for installing required dependencies
> (e.g., Helm). Once you've installed k0rdent in the management cluster
> and have kubectl, Helm, and other resources connected, you'll mostly
> be dealing with Kubernetes, and everything should work the same way on
> any host OS.

## Limitations

This QuickStart guides you in quickly creating a minimal k0rdent working environment. Setting up k0rdent for production is detailed in the [Administrator Guide](admin-before.md).

The current QuickStart focuses on AWS and Azure cloud environments, and guides in creating 'standalone' clusters &mdash; in k0rdent parlance, that means 'CNCF-certified Kubernetes clusters with control planes and workers hosted on cloud virtual machines.' The 'CNCF-certified Kubernetes cluster' is the [k0s Kubernetes distro](https://k0sproject.io).

k0rdent can do so much more today. Let's take a look at what's possible.

## Coming soon

QuickStarts for other Kubernetes distros, clouds, and environments will appear in the near future (short-term roadmap below):

* **AWS EKS hosted** &mdash; Amazon Elastic Kubernetes Service managed clusters 
* **Azure AKS hosted** &mdash; Azure Kubernetes Service
* **vSphere standalone** &mdash; k0s Kubernetes on vSphere virtual machines
* **OpenStack standalone** &mdash; k0s Kubernetes on OpenStack virtual machines

Plus (intermediate-term roadmap) tutorials for using k0rdent to create and manage hybrid, edge, and distributed platforms with Kubernetes-hosted control planes and workers on local or remote substrates.

**Demo/Tutorials:** We will also be converting the demos gradually into tutorials that explain how to use k0rdent for:

* Adding services to individual managed clusters, enabling management of complete platforms/IDPs
* Adding services to multiple managed clusters, enabling at-scale implementation and lifecycle management of standardized environments
* (As a Platform Architect) Authorizing cluster and service templates for use by others, and constraining their use within guardrails (enabling self-service)
* (As an authorized user) Leveraging shared cluster and service templates to lifecycle manage platforms (performing self-service)
* ... and more

Next you'll learn [how to use k0rdent](quickstart-1-mgmt-node-and-cluster.md).

