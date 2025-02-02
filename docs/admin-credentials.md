# The Credential System

In order for k0rdent to be able to take action on a particular provider, it must have the proper credentials.
This chapter explains how that system works.

## The process

In order to pass credentials to k0rdent so it can take action, the following has to happen:

1. The lead platform engineer or whoever has access to the actual provider credentials creates a `Secret` that includes that information. For example, for an AWS cluster, it might look like this:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
        name: aws-cluster-identity-secret
        namespace: kcm-system
    type: Opaque
    stringData:
        AccessKeyID: EXAMPLE_ACCESS_KEY_ID
        SecretAccessKey: EXAMPLE_SECRET_ACCESS_KEY
    ```

    Once this secret is created, it can be referenced without the user having access to the actual content, and thus the actual credentials.

2. A provider-specific `ClusterIdentity` gets created. The `ClusterIdentity` references the `Secret` from step one. For example, for an AWS cluster, this object might look like this:

	```yaml
	kind: AWSClusterStaticIdentity
	metadata:
	  name: aws-cluster-identity
	spec:
	  secretRef: aws-cluster-identity-secret
	  allowedNamespaces:
	    selector:
		  matchLabels: {}
	```

    Notice that it references the `aws-cluster-identity-secret` we created earlier. It also specifies the namespaces in which this `ClusterIdentity` can be used. (In this case there are no restrictions.)

2. Now you can create a `Credential` object that references the `ClusterIdentity`, thus making the credentials available and specifying the namespaces where it can be used. Continuing our AWS example:

	```yaml
	apiVersion: k0rdent.mirantis.com/v1alpha1
	kind: Credential
	metadata:
	  name: aws-cluster-credential
	  namespace: kcm-system
	spec:
  	  description: "Credential Example"
	  identityRef:
	    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
	    kind: AWSClusterStaticIdentity
	    name: aws-cluster-identity
	```
    Notice that it references the previous `ClusterIdentity` (in this case an `AWSClusterStaticIdentity`). Also notice that you can use the `.spec.description` field to add additional text about the `Credential` so users can choose if multiple `Credential`s are available.

3. Finally, when you create a `ClusterDeployment`, you reference the `Credential` object in order to enable k0rdent to pass that information to the infrastructure provider:

    ```yaml
	apiVersion: k0rdent.mirantis.com/v1alpha1
	kind: ClusterDeployment
	metadata:
      name: my-aws-clusterdeployment
	  namespace: kcm-system
	spec:
	  template: aws-standalone-cp-0-0-5
	  credential: aws-cluster-credential
      config:
	    clusterLabels: {}
	    region: us-east-2
	    controlPlane:
		  instanceType: t3.small
	    worker:
		  instanceType: t3.small
	```

    As you can see, the user doesn't have to pass anything but the name of the `Credential` in order to deploy the cluster. So all an administrator has to do is add these `Credential`s to the system and make them available. Note also that the `Credential` has to be available in the `ClusterDeployment`s namespace. (See [Cloud provider credentials propagation](#cloud-provider-credentials-propagation) for more information on how that works. )
    
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

## Cloud provider credentials propagation

Some components in the cluster deployment require cloud provider credentials to be
passed for proper functioning. As an example, Cloud Controller Manager (CCM)
requires provider credentials to create load balancers and provide other
functionality.

This poses a challenge of credentials delivery. Currently `cloud-init` is used
to pass all necessary credentials. This approach has several problems:

- Credentials stored unencrypted in the instance metadata.
- Rotation of the credentials is impossible without complete instance
  redeployment.
- Possible leaks, since credentials are copied to several `Secret` objects
  related to bootstrap data.

To solve these problems in k0rdent we're using the Sveltos controller, which can
render the CCM template with all necessary data from the CAPI provider resources (like
`ClusterIdentity`) and can create secrets directly on the cluster deployment.

> NOTE:
> CCM template examples can be found in `*-credentials.yaml` [here](https://github.com/k0rdent/kcm/tree/main/config/dev).
> Look for the `ConfigMap` object that has the `projectsveltos.io/template: "true"`
> annotation and `*-resource-template` as the object name.

This eliminates the need to pass anything credentials-related to `cloud-init`
and makes it possible to rotate credentials automatically without the need for
instance redeployment.

Also this automation makes it possible to separate roles and responsibilities
where only the lead engineer has access to credentials and other engineers can
use them without seeing values and even any access to underlying
infrastructure platform.

The process is fully automated and credentials will be propagated automatically
within the `ClusterDeployment` reconciliation process; user only needs to provide
the correct `Credential` object.

### Provider-specific notes

Since this feature depends on the provider, it's important to review any provider-specific notes and clarifications.

> NOTE: 
> More detailed research notes can be found [here](https://github.com/k0rdent/kcm/issues/293).

#### AWS

Since AWS uses roles, which are assigned to instances, no additional credentials
will be created.

The AWS provider supports 3 types of `ClusterIdentity` and, which one to use depends on
your specific use case. More information regarding CAPA `ClusterIdentity`
resources can be found in the [CRD Reference](https://cluster-api-aws.sigs.k8s.io/crd/).

#### Azure

Currently the Cluster API Azure (CAPZ) provider creates `azure.json` `Secret`s in the
same namespace as the `Cluster` object. By design they should be referenced in the
`cloud-init` YAML later during bootstrap process.

In k0rdent these Secrets aren't used and will not be added to the
`cloud-init`, but engineers can access them without restrictions, which is a security issue.

#### OpenStack

For OpenStack, CAPO relies on a `clouds.yaml` file.
In k0rdent, you provide this file in a Kubernetes `Secret` that references OpenStack credentials
(ideally application credentials for enhanced security). During reconciliation, kcm
automatically generates the cloud-config required by OpenStack’s cloud-controller-manager.

For more details, refer to the [kcm OpenStack Credential Propagation doc](https://github.com/k0rdent/kcm/blob/main/docs/dev.md#openstack).


#### Adopted clusters

Credentials for adopted clusters consist of a secret containing a kubeconfig file to access the existing kubernetes cluster. 
The kubeconfig file for the cluster should be contained in the value key of the secret object. The following is an example of 
a secret which contains the kubeconfig for an adopted cluster. To create this secret, first create or obtain a kubeconfig file 
for the cluster that is being adopted and then run the following command to base64 encode it:

```shell
cat kubeconfig | base64 -w 0
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
## The Credential Distribution System

k0rdent provides a mechanism to distribute `Credential` objects across namespaces using the
`AccessManagement` object. This object defines a set of `accessRules` that determine how credentials are distributed.

Each access rule specifies:

1. The target namespaces where credentials should be delivered.
2. A list of `Credential` names to distribute to those namespaces.

<!-- TODO show an example of the AccessManagement object. -->

The kcm controller copies the specified `Credential` objects from the `system` namespace to the target
namespaces based on the `accessRules` in the `AccessManagement` spec.

> INFO:
> Access rules can also include `Cluster` and `Service` TemplateChains (`clusterTemplateChains` and
> `serviceTemplateChains`) to distribute templates to target namespaces.
> For more details, read: [Template Life Cycle Management](template-intro.md).

### How to Configure Credential Distribution

To configure the distribution of `Credential` objects:

1. Edit the `AccessManagement` object.
2. Populate the `.spec.accessRules` field with the list of `Credential` names and the target namespaces.

Here’s an example configuration:

<!-- TODO show complete AccessManagement object. -->

```yaml
spec:
  accessRules:
  - targetNamespaces:
      list:
        - dev
        - test
    credentials:
      - aws-demo
      - azure-demo
```

In this example, the `aws-demo` and `azure-demo` `Credential` objects will be distributed to the `dev` and `test`
namespaces.


