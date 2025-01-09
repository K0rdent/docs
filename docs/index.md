# Welcome to Mirantis k0rdent Documentation

## Introduction

k0rdent is focused on developing a consistent way to deploy and manage
Kubernetes clusters at scale. One way to think of k0rdent is as a "super
control plane" designed to manage other Kubernetes control planes. Another
way to think of k0rdent is as a platform for Platform Engineering. If you
are building an internal developer platform (IDP), need a way to manage
Kubernetes clusters at scale in a centralized place, create Golden Paths,
etc. k0rdent is a great way to do that.

Whether you want to manage Kubernetes clusters on-premises, in the cloud,
or a combination of both, k0rdent provides a consistent way to do so. With
full life-cycle management, including provisioning, configuration, and
maintenance, k0rdent is designed to be a repeatable and secure way to
manage your Kubernetes clusters in a central location.

## k0rdent vs Project 2A vs HMC naming

k0rdent is the official name of an internal Mirantis project that was
originally codenamed "Project 2A". During our initial skunkworks-style
3-month MVP push, the code was put into a repository named HMC, which
stood for "Hybrid Multi-Cluster Controller". What is HMC became k0rdent
Cluster Manager (kcm), but it may be a little confusing because the
overall project was still called "Project 2A" or even "HMC" at times.

So, to be clear, here are the names and components:

- **k0rdent**: the overall project name
  - **k0rdent Cluster Manager (kcm)**
  - **k0rdent State Manager (ksm)**
    - This is currently rolled into kcm, but will be split out in the
      future
    - ksm leverages [Project Sveltos](https://github.com/projectsveltos/sveltos)
      for certain functionality
  - **k0rdent Observability and FinOps (kof)**
- **Project 2A**: the original codename of k0rdent (may occasionally show
  up in some documentation)
- **HMC or hmc**: the original repository name for k0rdent and kcm
  development (may occasionally show up in some documentation and code)
- **motel**: the original repository and codename for kof (may
  occasionally show up in some documentation and code)

## k0rdent Components

The main components of k0rdent include:

 * **k0rdent Cluster Manager (kcm)**

    Deployment and life-cycle management of Kubernetes clusters, including
    configuration, updates, and other CRUD operations.

 * **k0rdent State Manager (ksm)**

    Installation and life-cycle management of [beach-head services](glossary.md#beach-head-services),
    policy, Kubernetes API configurations and more.

 * **k0rdent Observability and FinOps (kof)**

    Cluster and beach-head services monitoring, events and log management.

## Quick Start

See the [k0rdent Quick Start Guide](quick-start/installation.md).

## Supported Providers

k0rdent leverages the Cluster API provider ecosystem, the following
providers have had `ProviderTemplates` created and validated, and more are
in the works.

 * [AWS](quick-start/aws.md)
 * [Azure](quick-start/azure.md)
 * [vSphere](quick-start/vsphere.md)
 * [OpenStack](quick-start/openstack.md)

## Development Documentation

Documentation related to development process and developer specific notes
located in the [main repository](https://github.com/k0rdent/kcm/blob/main/docs/dev.md).
