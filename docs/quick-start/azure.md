# Azure Quick Start

Much of the following includes the process of setting up credentials for Azure.
To better understand how k0rdent uses credentials, read the
[Credential System](../credential/main.md).

## Prerequisites

### k0rdent Management Cluster

You need a Kubernetes cluster with [kcm installed](installation.md).

### Software prerequisites

Before deploying Kubernetes clusters on Azure using k0rdent, ensure you have:

The Azure CLI (`az`) is required to interact with Azure resources. Install it
by following the [Azure CLI installation instructions](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

Run the `az login` command to authenticate your session with Azure.

### Register resource providers

If you're using a new subscription that has never been used to deploy kcm or
CAPI clusters, ensure the following resource providers are registered:

- `Microsoft.Compute`
- `Microsoft.Network`
- `Microsoft.ContainerService`
- `Microsoft.ManagedIdentity`
- `Microsoft.Authorization`

To register these providers, run the following commands in the Azure CLI:

```shell
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ManagedIdentity
az provider register --namespace Microsoft.Authorization
```

You can follow the [official documentation guide](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/resource-providers-and-types)
to register the providers.

Before creating a cluster on Azure, set up credentials. This involves creating
an `AzureClusterIdentity` and a _Service Principal (SP)_ to let CAPZ (Cluster
API Azure) communicate with Azure.

## Step 1: Find Your Subscription ID

List all your Azure subscriptions:

```shell
az account list -o table
```

Look for the Subscription ID of the account you want to use.

Example output:

```diff
Name                     SubscriptionId                        TenantId
-----------------------  -------------------------------------  --------------------------------
My Azure Subscription    12345678-1234-5678-1234-567812345678  87654321-1234-5678-1234-12345678
```

Copy your chosen Subscription ID for the next step.

## Step 2: Create a Service Principal (SP)

The Service Principal is like a password-protected user that CAPZ will use to
manage resources on Azure.

In your terminal, run the following command. Replace `<subscription-id>` with
the ID you copied earlier:

```bash
az ad sp create-for-rbac --role contributor --scopes="/subscriptions/<subscription-id>"
```

You will see output like this:

```json
{
 "appId": "12345678-7848-4ce6-9be9-a4b3eecca0ff",
 "displayName": "azure-cli-2024-10-24-17-36-47",
 "password": "12~34~I5zKrL5Kem2aXsXUw6tIig0M~3~1234567",
 "tenant": "12345678-959b-481f-b094-eb043a87570a"
}
```

> NOTE:
> Make sure to treat these strings like passwords. Do not share them
> or check them into a repository.

## Step 3: Create a Secret Object with the password

The Secret stores the `clientSecret` (password) from the Service Principal.

Save the Secret YAML into a file named `azure-cluster-identity-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: azure-cluster-identity-secret
  namespace: kcm-system
stringData:
  clientSecret: <password> # Password retrieved from the Service Principal
type: Opaque
```

Apply the YAML to your cluster using the following command:

```shell
kubectl apply -f azure-cluster-identity-secret.yaml
```

## Step 4: Create the AzureClusterIdentity Object

This object defines the credentials CAPZ will use to manage Azure resources.
It references the Secret you just created above.

> WARNING:
> Make sure that `.spec.clientSecret.name` matches the name of the
> Secret you created in the previous step.

Save the following YAML into a file named `azure-cluster-identity.yaml`:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AzureClusterIdentity
metadata:
  labels:
    clusterctl.cluster.x-k8s.io/move-hierarchy: "true"
  name: azure-cluster-identity
  namespace: kcm-system
spec:
  allowedNamespaces: {}
  clientID: <appId> # The App ID retrieved from the Service Principal above in Step 2
  clientSecret:
    name: azure-cluster-identity-secret
    namespace: kcm-system
  tenantID: <tenant> # The Tenant ID retrieved from the Service Principal above in Step 2
  type: ServicePrincipal
```

Apply the YAML to your cluster:

```shell
kubectl apply -f azure-cluster-identity.yaml
```

## Step 5: Create the kcm Credential Object

Create a YAML with the specification of our credential and save it as
`azure-cluster-identity-cred.yaml`.

> WARNING:
> `.spec.kind` must be `AzureClusterIdentity`  
> `.spec.name` must match `.metadata.name` of the `AzureClusterIdentity` object
> created in the previous step.

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Credential
metadata:
  name: azure-cluster-identity-cred
  namespace: kcm-system
spec:
  identityRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AzureClusterIdentity
    name: azure-cluster-identity
    namespace: kcm-system
```

Apply the YAML to your cluster:

```shell
kubectl apply -f azure-cluster-identity-cred.yaml
```

This creates the `Credential` object that will be used in the next step.

## Step 6: Create your first ClusterDeployment

Create a YAML with the specification of your managed Cluster and save it as
`my-azure-clusterdeployment1.yaml`.

Here is an example of a `ClusterDeployment` YAML file:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-azure-clusterdeployment1
  namespace: kcm-system
spec:
  template: azure-standalone-cp-0-0-3
  credential: azure-cluster-identity-cred
  config:
    location: "westus" # Select your desired Azure Location (find it via `az account list-locations -o table`)
    subscriptionID: <subscription-id> # Enter the Subscription ID used earlier
    controlPlane:
      vmSize: Standard_A4_v2
    worker:
      vmSize: Standard_A4_v2
```

Apply the YAML to your management cluster:

```shell
kubectl apply -f my-azure-clusterdeployment1.yaml
```

There will be a delay as the cluster finishes provisioning. Follow the
provisioning process with the following command:

```shell
kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com my-azure-clusterdeployment1 --watch
```

After the cluster is `Ready`, you can access it via the kubeconfig, like this:

```shell
kubectl -n kcm-system get secret my-azure-clusterdeployment1-kubeconfig -o jsonpath='{.data.value}' | base64 -d > my-azure-clusterdeployment1-kubeconfig.kubeconfig
```

```shell
KUBECONFIG="my-azure-clusterdeployment1-kubeconfig.kubeconfig" kubectl get pods -A
```
