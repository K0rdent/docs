# Azure machine parameters

## SSH

The SSH public key can be passed to `.spec.config.sshPublicKey` 
parameter (in the case of a hosted control plane) or `.spec.config.controlPlane.sshPublicKey` and
`.spec.config.worker.sshPublicKey` parameters (in the case of a standalone control plane)
of the `ClusterDeployment` object.

It should be encoded in **base64** format.

## VM size

Azure supports various VM sizes which can be retrieved with the following
command:

```bash
az vm list-sizes --location "<location>" -o table
```

Then desired VM size could be passed to the:

- `.spec.config.vmSize` - for hosted CP deployment.
- `.spec.config.controlPlane.vmSize` - for control plane nodes in the standalone
  deployment.
- `.spec.config.worker.vmSize` - for worker nodes in the standalone deployment.

*Example: Standard_A4_v2*

## Root Volume size

Root volume size of the VM (in GB) can be changed through the following
parameters:

- `.spec.config.rootVolumeSize` - for hosted CP deployment.
- `.spec.config.controlPlane.rootVolumeSize` - for control plane nodes in the
  standalone deployment.
- `.spec.config.worker.rootVolumeSize` - for worker nodes in the standalone
  deployment.

*Default value: 30*

Please note that this value can't be less than size of the root volume 
defined in your image.

## VM Image

You can define the image which will be used for your machine using the following
parameters:

- `.spec.config.image` - for hosted CP deployment.
- `.spec.config.controlPlane.image` - for control plane nodes in the standalone
  deployment.
- `.spec.config.worker.image` - for worker nodes in the standalone deployment.

There are multiple self-excluding ways to define the image source (for example Azure
Compute Gallery, Azure Marketplace, and so on).

Detailed information regarding image can be found in [CAPZ documentation](https://capz.sigs.k8s.io/self-managed/custom-images)

By default, the latest official CAPZ Ubuntu based image is used.

# Azure Hosted control plane (k0smotron) deployment

## Prerequisites

-   Management Kubernetes cluster (v1.28+) deployed on Azure with kcm installed
    on it
-   Default storage class configured on the management cluster

Keep in mind that all control plane components for all managed clusters will
reside in the management cluster.

## Pre-existing resources

Certain resources will not be created automatically in a hosted control plane
scenario, so you must create them in advance and provide them to  the `ClusterDeployment`
object. You can reuse these resources with management cluster as described
below.

If you deployed your Azure Kubernetes cluster using Cluster API Provider Azure
(CAPZ) you can obtain all the necessary data with the commands below:

**Location**

```bash
kubectl get azurecluster <cluster-name> -o go-template='{{.spec.location}}'
```

**Subscription ID**

```bash
kubectl get azurecluster <cluster-name> -o go-template='{{.spec.subscriptionID}}'
```

**Resource group**

```bash
kubectl get azurecluster <cluster-name> -o go-template='{{.spec.resourceGroup}}'
```

**vnet name**

```bash
kubectl get azurecluster <cluster-name> -o go-template='{{.spec.networkSpec.vnet.name}}'
```

**Subnet name**

```bash
kubectl get azurecluster <cluster-name> -o go-template='{{(index .spec.networkSpec.subnets 1).name}}'
```

**Route table name**

```bash
kubectl get azurecluster <cluster-name> -o go-template='{{(index .spec.networkSpec.subnets 1).routeTable.name}}'
```

**Security group name**

```bash
kubectl get azurecluster <cluster-name> -o go-template='{{(index .spec.networkSpec.subnets 1).securityGroup.name}}'
```



## kcm ClusterDeployment manifest

With all the collected data your `ClusterDeployment` manifest will look similar to this:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: azure-hosted-cp
spec:
  template: azure-hosted-cp-0-0-2
  credential: azure-credential
  config:
    clusterLabels: {}
    location: "westus"
    subscriptionID: SUBSCRIPTION_ID_SUBSCRIPTION_ID
    vmSize: Standard_A4_v2
    resourceGroup: mgmt-cluster
    network:
      vnetName: mgmt-cluster-vnet
      nodeSubnetName: mgmt-cluster-node-subnet
      routeTableName: mgmt-cluster-node-routetable
      securityGroupName: mgmt-cluster-node-nsg
```

To simplify creation of the ClusterDeployment object you can use the template below:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: azure-hosted-cp
spec:
  template: azure-hosted-cp-0-0-2
  credential: azure-credential
  config:
    clusterLabels: {}
    location: "{{.spec.location}}"
    subscriptionID: "{{.spec.subscriptionID}}"
    vmSize: Standard_A4_v2
    resourceGroup: "{{.spec.resourceGroup}}"
    network:
      vnetName: "{{.spec.networkSpec.vnet.name}}"
      nodeSubnetName: "{{(index .spec.networkSpec.subnets 1).name}}"
      routeTableName: "{{(index .spec.networkSpec.subnets 1).routeTable.name}}"
      securityGroupName: "{{(index .spec.networkSpec.subnets 1).securityGroup.name}}"
```

Then you can render it using the command:

```bash
kubectl get azurecluster <management-cluster-name> -o go-template="$(cat template.yaml)"
```

## Cluster creation

After applying the `ClusterDeployment` object you must manually set the status of
the `AzureCluster` object due to limitations in k0smotron (see
[k0sproject/k0smotron#668](https://github.com/k0sproject/k0smotron/issues/668)).

Execute the following command:

```bash
kubectl patch azurecluster <cluster-name> --type=merge --subresource status --patch 'status: {ready: true}'
```

## Important notes on the cluster deletion

Because of the aforementioned limitation you also need to take manual steps in
order to properly delete an Azurecluster.

Before removing the cluster, make sure to place a custom finalizer on the
`AzureCluster` object. This is needed to prevent it from being deleted instantly,
which will cause cluster deletion to stuck indefinitely.

To place a finalizer you can execute the following command:

```bash
kubectl patch azurecluster <cluster-name> --type=merge --patch 'metadata: {finalizers: [manual]}'
```

When the finalizer is placed you can remove the `ClusterDeployment` as usual. Check that
all `AzureMachine` objects are deleted successfully, and remove the finalizer you've
placed to finish cluster deletion.

If you have orphaned `AzureMachine`s left, you'll have to delete the finalizers on
them manually after making sure that no VMs are present in Azure.

> NOTE:
> Since Azure admission prohibits orphaned objects mutation, you'll have to disable
> it by deleting its `mutatingwebhookconfiguration`.