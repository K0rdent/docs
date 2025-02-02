# Welcome to the k0rdent docs

## Introduction

k0rdent has been developed to provide a way to manage distributed infrastructure
at massive scale leveraging kubernetes.

The project is based on the premise that:

* Kubernetes and its ecosystem is mature and inherently stable.
* Large scale adoption means that it can run anywhere.
* Community standards and open source projects ensure support and reduces adoption risk.

The goal of the k0rdent project is to provide platform engineers with the means
to deliver a distributed container management environment (DCME) and enable them to
compose unique internal developer platforms (IDP) to support a diverse range
of complex modern application workloads.

Another way to think of k0rdent is as a "super control plane" designed to ensure the
consistent provisioning and lifecycle management of kubernetes clusters and the
services that make it useful.

In short:
Kubernetes clusters at scale, managed centrally, template driven, based on open
community driven standards, enabling Golden Paths, k0rdent aspires to do that.

Whether you want to manage Kubernetes clusters on-premises, in the cloud,
or a combination of both, k0rdent provides a consistent way to do so. With
full life-cycle management, including provisioning, configuration, and
maintenance, k0rdent is designed to be a repeatable and secure way to
manage your Kubernetes clusters in a central location.

## k0rdent vs Project 2A

k0rdent is the official name of this open source project developed by Mirantis, it
started out life as and internal project that we codenamed "Project 2A". The 2A
referencing the hexadecimal 0x2A, which encompassed our hopes for the project.

## Project Components

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

## Structure and History

The project has a number of components, here are the names and components:

* **k0rdent**: the overall project name
  * **k0rdent Cluster Manager (kcm)**
  * **k0rdent State Manager (ksm)**
    * This is currently rolled into kcm, but may be split out in the
      future
    * ksm leverages [Project Sveltos](https://github.com/projectsveltos/sveltos)
      for and increasing amount of functionality
  * **k0rdent Observability and FinOps (kof)**

There are a few historical names that may show up in the code and older docs.

* **Project 2A**: the original codename of k0rdent (may occasionally show
  up in some documentation)
* **HMC or hmc**: the original repository name for k0rdent and kcm
  development (may occasionally show up in some documentation and code)
* **motel**: the original repository and codename for kof (may
  occasionally show up in some documentation and code)

## Quick Start

See the [k0rdent Quick Start Guide](guide-to-quickstarts.md).

## Supported Providers

k0rdent leverages the Cluster API provider ecosystem, the following
providers have had `ProviderTemplates` created and validated, and more are
in the works. k0rdent also includes a way to add custom providers.

* [AWS](admin-prepare.md#aws)
* [Azure](admin-prepare.md#azure)
* [vSphere](admin-prepare.md#vsphere)
* [OpenStack](admin-prepare.md#openstack)

## Development Documentation

Documentation related to development process and developer specific notes
located in the [main repository](https://github.com/k0rdent/kcm/blob/main/docs/dev.md).