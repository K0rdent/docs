# Installation Guide

This section describes how to install Project 2A.

## TL;DR

```bash
export KUBECONFIG=<path-to-management-kubeconfig>
```

```bash
helm install hmc oci://ghcr.io/k0rdent/kcm/charts/hmc --version <hmc-version> -n hmc-system --create-namespace
```

This will use the defaults as seen in Extended Management Configuration section below.

## Finding Releases

Releases are tagged in the GitHub repository and can be found [here](https://github.com/k0rdent/kcm/tags).

## Extended Management Configuration

Project 2A is deployed with the following default configuration, which may vary
depending on the release version:

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: Management
metadata:
  name: hmc
spec:
  providers:
  - name: k0smotron
  - name: cluster-api-provider-aws
  - name: cluster-api-provider-azure
  - name: cluster-api-provider-vsphere
  - name: projectsveltos
  release: hmc-0-0-6
```
To see what is included in a specific release, look at the `release.yaml` file in the tagged release.
For example, here is the [v0.0.6 release.yaml](https://github.com/k0rdent/kcm/releases/download/v0.0.6/release.yaml).

There are two options to override the default management configuration of Project 2A:

1. Update the `Management` object after the Project 2A installation using `kubectl`:

    `kubectl --kubeconfig <path-to-management-kubeconfig> edit management`

2. Deploy 2A skipping the default `Management` object creation and provide your
   own `Management` configuration:

	- Create `management.yaml` file and configure core components and providers.
	- Specify `--create-management=false` controller argument and install Project 2A:
	  If installing using `helm` add the following parameter to the `helm
	  install` command:

		  `--set="controller.createManagement=false"`

	- Create `hmc` `Management` object after Project 2A installation:

           ```bash
           kubectl --kubeconfig <path-to-management-kubeconfig> create -f management.yaml
           ```

## Air-gapped installation

Follow the [Air-gapped Installation Guide](airgap.md) to get the instructions on
how to perform 2A installation in the air-gapped environment.

## Cleanup

1. Remove the Management object:

  ```bash
	kubectl delete management.hmc hmc
  ```

> WARNING: 
> 
> Make sure you have no Project 2A `ManagedCluster` objects left in the cluster prior to deletion.

2. Remove the `hmc` Helm release:

  ```bash
	helm uninstall hmc -n hmc-system
  ```

3. Remove the `hmc-system` namespace:

  ```bash
	kubectl delete ns hmc-system
  ```
