# Adopting existing Kubernetes cluster guide

## Prerequisites

In order to adopt an existing Kubernetes cluster you will require the following:

- A kubernetes kubeconfig file for the cluster to be adopted
- A management cluster with K0rdent installed
- Network connectivity between the management cluster and the cluster to be adopted

## Installation


### Step 1: Create the credential
Create a `Credential` object with all credentials required per the
  [Credential System](../credential/main.md).

### Step 2: Configure the adopted cluster template

- Set the `KUBECONFIG` environment variable to the path to the management
  cluster kubeconfig file.

### Step 3: Create the ClusterDeployment Object YAML Configuration

- Create the file with the `ClusterDeployment` configuration:

    ```yaml
    apiVersion: hmc.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: <cluster-name>
      namespace: <hmc-system-namespace>
    spec:
      template: adopted-cluster-<template-version>
      credential: <credential-name>
      dryRun: <"true" or "false": defaults to "false">
      config:
        <cluster-configuration>
    ```

> NOTE:
>
> Substitute the parameters enclosed in angle brackets with the corresponding
> values. Enable the `dryRun` flag if required. For details, see
> [Dry Run](#dry-run).

Following is an interpolated example.

> EXAMPLE: `ClusterDeployment` 
>
> ```yaml
> apiVersion: hmc.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   name: my-cluster
>   namespace: hmc-system
> spec:
>   template: adotped-cluster-0-0-1
>   credential: my-cluster-credential
>   dryRun: true
> ```


### Step 4: Apply the `ClusterDeployment` Configuration to Create it

- Apply the `ClusterDeployment` object to your Project 2A deployment:

  ```shell
  kubectl apply -f clusterdeployment.yaml
  ```

### Step 5: Check the Status of the `ClusterDeployment` Object

- Check the status of the newly created `ClusterDeployment`:

  ```shell
  kubectl -n <namespace> get clusterdeployment.hmc <cluster-name> -o=yaml
  ```
