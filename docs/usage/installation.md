# Installation Guide

This section describes how to install k0rdent.

## TL;DR

```bash
export KUBECONFIG=<path-to-management-kubeconfig>
```

```bash
helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm --version <kcm-version> -n kcm-system --create-namespace
```

This will use the defaults as seen in Extended Management Configuration section below.

## Finding Releases

Releases are tagged in the GitHub repository and can be found [here](https://github.com/k0rdent/kcm/tags).

## Extended Management Configuration

k0rdent is deployed with the following default configuration, which may vary
depending on the release version:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Management
metadata:
  name: kcm
spec:
  providers:
  - name: k0smotron
  - name: cluster-api-provider-aws
  - name: cluster-api-provider-azure
  - name: cluster-api-provider-vsphere
  - name: projectsveltos
release: kcm-0-0-7
```
To see what is included in a specific release, look at the `release.yaml` file in the tagged release.
For example, here is the [v0.0.7 release.yaml](https://github.com/k0rdent/kcm/releases/download/v0.0.7/release.yaml).

There are two options to override the default management configuration of k0rdent:

1. Update the `Management` object after the k0rdent installation using `kubectl`:

    `kubectl --kubeconfig <path-to-management-kubeconfig> edit management`

2. Deploy k0rdent skipping the default `Management` object creation and provide your
   own `Management` configuration:

	- Create `management.yaml` file and configure core components and providers.
	- Specify `--create-management=false` controller argument and install k0rdent:
	  If installing using `helm` add the following parameter to the `helm
	  install` command:

		  `--set="controller.createManagement=false"`

	- Create `kcm` `Management` object after k0rdent installation:

           ```bash
           kubectl --kubeconfig <path-to-management-kubeconfig> create -f management.yaml
           ```

## Air-gapped installation

Follow the [Air-gapped Installation Guide](airgap.md) to get the instructions on
how to perform k0rdent installation in the air-gapped environment.

## Cleanup

1. Remove the Management object:

  ```bash
	kubectl delete management.kcm kcm
  ```

> WARNING: 
> 
> Make sure you have no k0rdent `ClusterDeployment` objects left in the cluster prior to deletion.

2. Remove the `kcm` Helm release:

  ```bash
	helm uninstall kcm -n kcm-system
  ```

3. Remove the `kcm-system` namespace:

  ```bash
	kubectl delete ns kcm-system
  ```
