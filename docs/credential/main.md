# Credential System

In order for infrastructure provider to work properly a correct credentials
should be passed to it. The following describes how it is implemented in k0rdent.

## The process

The following is the process of passing credentials to the system:

1. Provider specific `ClusterIdentity` and `Secret` are created
2. `Credential` object is created referencing `ClusterIdentity` from step **1**.
3. The `Credential` object is then referenced in the `ClusterDeployment`.
4. Optionally, certain credentials MAY be propagated to the `ClusterDeployment` after it is created.

The following diagram illustrates the process:

```mermaid
flowchart TD
  Step1["<b>Step 1</b> (Lead Engineer):<br/>Create ClusterIdentity and Secret objects where ClusterIdentity references Secret"]
  Step1 --> Step2["<b>Step 2</b> (Any Engineer):<br/>Create Credential object referencing ClusterIdentity"]
  Step2 --> Step3["<b>Step 3</b> (Any Engineer):<br/>Create ClusterDeployment referencing Credential object"]
  Step3 --> Step4["<b>Step 4</b> (Any Engineer):<br/>Apply ClusterDeployment, wait for provisioning & reconciliation, then propagate credentials to nodes if necessary"]
```

By design steps 1 and 2 should be executed by the lead engineer who has
access to the credentials. Thus credentials could be used by engineers
without a need to have access to actual credentials or underlying resources,
like `ClusterIdentity`.

## Credential object

The `Credential` object acts like a reference to the underlying credentials. It
is namespace-scoped, which means that it must be in the same `Namespace` with
the `ClusterDeployment` it is referenced in. Actual credentials can be located in
any namespace.

### Example

```yaml
---
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Credential
metadata:
  name: azure-credential
  namespace: dev
spec:
  description: "Main Azure credentials"
  identityRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AzureClusterIdentity
    name: azure-cluster-identity
    namespace: kcm-system
```

In the example above `Credential` object is referencing `AzureClusterIdentity`
which was created in the `kcm-system` namespace.

The `.spec.description` field can be used to provide arbitrary description of the
object, so user could make a decision which credentials to use if several are
present.

## Cloud provider credentials propagation

Some components in the cluster deployment require cloud provider credentials to be
passed for proper functioning. As an example Cloud Controller Manager (CCM)
requires provider credentials to create load balancers and provide other
functionality.

This poses a challenge of credentials delivery. Currently `cloud-init` is used
to pass all necessary credentials. This approach has several problems:

- Credentials stored unencrypted in the instance metadata.
- Rotation of the credentials is impossible without complete instance
  redeployment.
- Possible leaks, since credentials are copied to several `Secret` objects
  related to bootstrap data.

To solve these problems in k0rdent we're using special controller which
aggregates all necessary data from CAPI provider resources (like
`ClusterIdentity`) and creates secrets directly on the cluster deployment.

This eliminates the need to pass anything credentials-related to `cloud-init`
and makes it possible to rotate credentials automatically without the need for
instance redeployment.

Also this automation makes it possible to separate roles and responsibilities
where only the lead engineer has access to credentials and other engineers can
use them without seeing values and even any access to underlying
infrastructure platform.

The process is fully automated and credentials will be propagated automatically
within the `ClusterDeployment` reconciliation process, user only needs to provide
the correct [Credential object](#credential-object).

### Provider specific notes

Since this feature depends on the provider some notes and clarifications
are needed for each provider.

> NOTE: 
> More detailed research notes can be found [here](https://github.com/k0rdent/kcm/issues/293).

#### AWS

Since AWS uses roles, which are assigned to instances, no additional credentials
will be created.

AWS provider supports 3 types of `ClusterIdentity`, which one to use depends on
your specific use case. More information regarding CAPA `ClusterIdentity`
resources could be found in [CRD Reference](https://cluster-api-aws.sigs.k8s.io/crd/).

#### Azure

Currently Cluster API provider Azure (CAPZ) creates `azure.json` Secrets in the
same namespace with `Cluster` object. By design they should be referenced in the
`cloud-init` YAML later during bootstrap process.

In k0rdent these Secrets aren't used and will not be added to the
`cloud-init`, but engineers can access them unrestricted.

#### OpenStack

For OpenStack, CAPO relies on a clouds.yaml file.
In k0rdent, you provide this file in a Kubernetes Secret that references OpenStack credentials
(ideally application credentials for enhanced security). During reconciliation, kcm
automatically generates the cloud-config required by OpenStack’s cloud-controller-manager.

For more details, refer to the [kcm OpenStack Credential Propagation doc](https://github.com/k0rdent/kcm/blob/main/docs/dev.md#openstack).


#### Adopted cluster

Credentials for adopted clusters consist of a secret containing a kubeconfig file to access the existing kubernetes cluster. 
The kubeconfig file for the cluster should be contained in the value key of the secret object. The following is an example of 
a secret which contains the kubeconfig for an adopted cluster. To create this secret, first create or obtain a kubeconfig file 
for the cluster that is being adopted and then run the following command to base64 encode it:

```shell
cat kubeconfig | base64 -d -w 0
```

Once you have obtained a base64 encoded kubeconfig file create a secret:

```yaml
apiVersion: v1
data:
  value: <base64 encoded kubeconfig file>
kind: Secret
metadata:
  name: adopted-cluster-kubeconf
  namespace: <namespace>
type: Opaque
```
