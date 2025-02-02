# QuickStart 2 - AWS target environment

In this QuickStart unit, we'll be gathering information and performing preparatory steps to enable k0rdent (running on your management node) to manage clusters on Amazon Web Services (AWS), and deploying our first managed cluster.

As noted in the [Guide to QuickStarts](guide-to-quickstarts.md), you'll need administrative access to AWS account to complete this step. If you haven't yet created a management node and installed k0rdent, go back to [QuickStart 1 - Management node and cluster](quickstart-1-mgmt-node-and-cluster.md).

Note that if you have already done our Azure QuickStart ([QuickStart 2 - Azure target environment](quickstart-2-azure.md)) you can continue here with steps to add the ability to manage clusters on AWS. The k0rdent management cluster can accommodate multiple provider and credential setups, enabling management of multiple infrastructures. And even if your management node is external to AWS (for example, it could be on an Azure virtual machine), as long as you permit outbound traffic to all IP addresses from the management node, this should work fine. A big benefit of k0rdent is that it provides a single point of control and visibility across multiple clusters on multiple clouds and infrastructures.

> NOTE:
> **CLOUD SECURITY 101**: k0rdent requires _some_ but not _all_ permissions
> to manage AWS &mdash; doing so via the CAPA (ClusterAPI for AWS) provider.
> So a best practice for using k0rdent with AWS (this pattern is repeated
> with other clouds and infrastructures) is to create a new 'k0rdent user'
> on your account with the particular permissions k0rdent and CAPA require.

In this section, we'll create and configure IAM for that user, and perform other steps to make that k0rdent user's credentials accessible to k0rdent in the management node.

> NOTE:
> If you're working on a shared AWS account, please ensure that the k0rdent user is not already set up before creating a new one.

Creating a k0rdent user with minimal required permissions is one of several principle-of-least-privilege mechanisms used to help ensure security as organizations work with k0rdent at progressively greater scales. For more on k0rdent security best practices, please see the [Administrator Guide]().

## Install the AWS CLI

We'll use the AWS CLI to create and set IAM permissions for the k0rdent user, so we'll install it on our management node:

```shell
sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" 
unzip awscliv2.zip 
sudo ./aws/install
```

## Install clusterawsadm

k0rdent uses Cluster API (CAPI) to marshal clouds and infrastructures. For AWS, this means using the components from the Cluster API Provider AWS (CAPA) project. This QuickStart leverages clusterawsadm, a CLI tool created by CAPA project that helps with AWS-specific tasks like IAM role, policy, and credential configuration.

To install clusterawsadm on Ubuntu on x86 hardware:

```shell
curl -LO https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases/download/v2.7.1/clusterawsadm-linux-amd64
sudo install -o root -g root -m 0755 clusterawsadm-linux-amd64 /usr/local/bin/clusterawsadm
```

## Export your administrative credentials

You should have these already, preserved somewhere safe. If not, you can visit the AWS webUI (Access Management > Users) and generate new credentials (Access Key ID, Secret Access Key, and Session Token (if using multi-factor authentication)).

Export the credentials to the management node environment:

```shell
export AWS_REGION=EXAMPLE_AWS_REGION
export AWS_ACCESS_KEY_ID=EXAMPLE_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=EXAMPLE_SECRET_ACCESS_KEY
export AWS_SESSION_TOKEN=EXAMPLE_SESSION_TOKEN # Optional. If you are using Multi-Factor Auth.
```

These credentials will be used both by the AWS CLI (to create your k0rdent user) and by clusterawsadm (to create a CloudFormation template used by CAPA within k0rdent).

## Create the k0rdent AWS user

Now we can use the AWS CLI to create a new k0rdent user:

```shell
 aws iam create-user --user-name k0rdentQuickstart
```

You'll see something like what's shown below. You should save this data securely. Note the Amazon Resource Name (ARN) because we'll be using it right away:

```console
{
    "User": {
        "Path": "/",
        "UserName": "k0rdentQuickstart",
        "UserId": "EXAMPLE_USER_ID",
        "Arn": "arn:aws:iam::FAKE_ARN_123:user/k0rdentQuickstart",
        "CreateDate": "2025-01-18T08:15:27+00:00"
    }
}
```

## Attach IAM policies to the k0rdent user

Next, we'll attach appropriate policies to the k0rdent user. These are:

* `control-plane.cluster-api-provider-aws.sigs.k8s.io`
* `controllers.cluster-api-provider-aws.sigs.k8s.io`
* `nodes.cluster-api-provider-aws.sigs.k8s.io`

We use the AWS CLI to attach them. To do this, you will need to extract the Amazon Resource Name (ARN) for the newly-created user. In the above example of content returned from the AWS CLI on user creation (see above) that's marked with the placeholder `FAKE_ARN_123`:

Given this, you can assemble and execute the following commands to implement the required policies:

```shell
aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::FAKE_ARN_123:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io
aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::FAKE_ARN_123:policy/controllers-eks.cluster-api-provider-aws.sigs.k8s.io
aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::FAKE_ARN_123:policy/controllers.cluster-api-provider-aws.sigs.k8s.io
```

We can check to see that policies were assigned:

```shell
aws iam list-policies --scope Local
```
And you'll see output that looks like this (this is non-valid example text):

```console
{
    "Policies": [
        {
            "PolicyName": "controllers-eks.cluster-api-provider-aws.sigs.k8s.io",
            "PolicyId": "ANPA22CF4NNF3VUDTMH3N",
            "Arn": "arn:aws:iam::FAKE_ARN_123:policy/controllers-eks.cluster-api-provider-aws.sigs.k8s.io",
            "Path": "/",
            "DefaultVersionId": "v1",
            "AttachmentCount": 2,
            "PermissionsBoundaryUsageCount": 0,
            "IsAttachable": true,
            "CreateDate": "2025-01-01T18:47:43+00:00",
            "UpdateDate": "2025-01-01T18:47:43+00:00"
        },
        {
            "PolicyName": "nodes.cluster-api-provider-aws.sigs.k8s.io",
            "PolicyId": "ANPA22CF4NNF5TAKL44PU",
            "Arn": "arn:aws:iam::FAKE_ARN_123:policy/nodes.cluster-api-provider-aws.sigs.k8s.io",
            "Path": "/",
            "DefaultVersionId": "v1",
            "AttachmentCount": 3,
            "PermissionsBoundaryUsageCount": 0,
            "IsAttachable": true,
            "CreateDate": "2025-01-01T18:47:44+00:00",
            "UpdateDate": "2025-01-01T18:47:44+00:00"
        },
        {
            "PolicyName": "controllers.cluster-api-provider-aws.sigs.k8s.io",
            "PolicyId": "ANPA22CF4NNFVO6OHIQOE",
            "Arn": "arn:aws:iam::FAKE_ARN_123:policy/controllers.cluster-api-provider-aws.sigs.k8s.io",
            "Path": "/",
            "DefaultVersionId": "v1",
            "AttachmentCount": 3,
            "PermissionsBoundaryUsageCount": 0,
            "IsAttachable": true,
            "CreateDate": "2025-01-01T18:47:43+00:00",
            "UpdateDate": "2025-01-01T18:47:43+00:00"
        },
        {
            "PolicyName": "control-plane.cluster-api-provider-aws.sigs.k8s.io",
            "PolicyId": "ANPA22CF4NNFY4FJ3DA2E",
            "Arn": "arn:aws:iam::FAKE_ARN_123:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io",
            "Path": "/",
            "DefaultVersionId": "v1",
            "AttachmentCount": 2,
            "PermissionsBoundaryUsageCount": 0,
            "IsAttachable": true,
            "CreateDate": "2025-01-01T18:47:43+00:00",
            "UpdateDate": "2025-01-01T18:47:43+00:00"
        }
    ]
}
```

## Create AWS credentials for the k0rdent user

In the AWS IAM Console, you can now create the Access Key ID and Secret Access Key for the k0rdent user and download them. You can also do this via the AWS CLI:

```shell
aws iam create-access-key --user-name k0rdentQuickstart
```

You should see something like this. It's important to save these credentials securely somewhere other than the management node, since the management node may end up being ephemeral. Again, this is non-valid example text:

```console
{
    "AccessKey": {
        "UserName": "k0rdentQuickstart",
        "AccessKeyId": "EXAMPLE_ACCESS_KEY_ID",
        "Status": "Active",
        "SecretAccessKey": "EXAMPLE_SECRET_ACCESS_KEY",
        "CreateDate": "2025-01-18T08:33:35+00:00"
    }
}
```

<!--
> WARNING:
> You may encounter an issue where EKS machines are not created due to the `ControlPlaneIsStable` preflight check
> failure during EKS cluster deployment. Please follow the
> [instruction](known-issues-eks.md#eks-machines-are-not-created-controlplaneisstable-preflight-check-failed)
> to apply the workaround.
-->

## Configure AWS IAM for k0rdent

Before k0rdent CAPI can manage resources on AWS, you need to prepare for this by using `clusterawsadm` to create a bootstrap CloudFormation stack with additional IAM policies and a service account. You do this under the administrative account credentials you earlier exported to the management node environment:

```shell
clusterawsadm bootstrap iam create-cloudformation-stack
```

## Create IAM credentials secret on the management cluster

Next, we create a secret containing credentials for the k0rdent user and apply this to the management cluster running k0rdent, in the kcm-system namespace (important: if you use another namespace, k0rdent will be unable to read the credentials). To do this, create the following YAML in a file called `aws-cluster-identity-secret.yaml':

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-cluster-identity-secret
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
type: Opaque
stringData:
  AccessKeyID: "EXAMPLE_ACCESS_KEY_ID"
  SecretAccessKey: "EXAMPLE_SECRET_ACCESS_KEY"
```

Remember: the Access Key ID and Secret Access Key are the ones you generated for the k0rdent user, `k0rdentQuickStart`.

Then apply this YAML to the management cluster as follows:

```shell
kubectl apply -f aws-cluster-identity-secret.yaml -n kcm-system
```

## Create the AWSClusterStaticIdentity object

Next, we need to create an `AWSClusterStaticIdentity` object that uses the secret.

To do this, create a YAML file named `aws-cluster-identity` as follows:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSClusterStaticIdentity
metadata:
  name: aws-cluster-identity
  labels:
    k0rdent.mirantis.com/component: "kcm"
spec:
  secretRef: aws-cluster-identity-secret
  allowedNamespaces:
    selector:
      matchLabels: {}
```

Note that the `spec.secretRef` is the same as the `metadata.name` of the secret we just created.

Create the object as follows:

```shell
kubectl apply -f aws-cluster-identity.yaml  -n kcm-system
```

## Create the k0rdent Cluster Manager credential object

Now we create the k0rdent Cluster Manager credential object. As in prior steps, create a YAML file called `aws-cluster-identity-cred.yaml':

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Credential
metadata:
  name: aws-cluster-identity-cred
  namespace: kcm-system
spec:
  description: "Credential Example"
  identityRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSClusterStaticIdentity
    name: aws-cluster-identity
```

Note that `.spec.identityRef.kind` must be `AWSClusterStaticIdentity` and `.spec.identityRef.name` must match the `.metadata.name` of the `AWSClusterStaticIdentity` object.

Now apply this YAML to your management cluster:

```shell
kubectl apply -f aws-cluster-identity-cred.yaml -n kcm-system
```

## List available cluster templates

k0rdent is now fully configured to manage AWS. To create a cluster, begin by listing the available `ClusterTemplate`s provided with k0rdent:

```shell
kubectl get clustertemplate -n kcm-system
```

You'll see output resembling what's below. Grab the name of the AWS standalone cluster template in its present version (in the below example, that's `aws-standalone-cp-0-0-5`):

```console
NAMESPACE    NAME                            VALID
kcm-system   adopted-cluster-0-0-2           true
kcm-system   aws-eks-0-0-3                   true
kcm-system   aws-hosted-cp-0-0-4             true
kcm-system   aws-standalone-cp-0-0-5         true
kcm-system   azure-aks-0-0-2                 true
kcm-system   azure-hosted-cp-0-0-4           true
kcm-system   azure-standalone-cp-0-0-5       true
kcm-system   openstack-standalone-cp-0-0-2   true
kcm-system   vsphere-hosted-cp-0-0-5         true
kcm-system   vsphere-standalone-cp-0-0-5     true
```

## Create your ClusterDeployment

Now, to deploy a cluster, create a YAML file called `my-aws-clusterdeployment1.yaml`. We'll use this to create a `ClusterDeployment` object in k0rdent, representing the deployed cluster. The `ClusterDeployment` identifies for k0rdent the `ClusterTemplate` you wish to use for cluster creation, the identity credential object you wish to create it under (that of your k0rdent user), plus the region and instance types you want to use to host control plane and worker nodes:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-aws-clusterdeployment1
  namespace: kcm-system
spec:
  template: aws-standalone-cp-0-0-5
  credential: aws-cluster-identity-cred
  config:
    clusterLabels: {}
    region: us-east-2
    controlPlane:
      instanceType: t3.small
    worker:
      instanceType: t3.small
```

## Apply the `ClusterDeployment` to deploy the cluster

Finally, we'll apply the `ClusterDeployment` YAML (`my-aws-clusterdeployment1.yaml`) to instruct k0rdent to deploy the cluster:

```shell
kubectl apply -f my-aws-clusterdeployment1.yaml
```

Kubernetes should confirm this:

```console
clusterdeployment.k0rdent.mirantis.com/my-aws-clusterdeployment1 created
```

There will be a delay as the cluster finishes provisioning. You can watch the provisioning process with the following command:

```shell
kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com my-aws-clusterdeployment1 --watch
```

In a short while, you'll see output such as:

```console
NAME                        READY   STATUS
my-aws-clusterdeployment1   True    ClusterDeployment is ready
```

## Obtain the cluster's kubeconfig

Now you can retrieve the cluster's `kubeconfig`:

```shell
kubectl -n kcm-system get secret my-aws-clusterdeployment1-kubeconfig -o jsonpath='{.data.value}' | base64 -d > my-aws-clusterdeployment1-kubeconfig.kubeconfig
```

And you can use the `kubeconfig` to see what's running on the cluster:

```shell
KUBECONFIG="my-aws-clusterdeployment1-kubeconfig.kubeconfig"
kubectl get pods -A
```

## List managed clusters

To verify the presence of the managed cluster, list the available `ClusterDeployment`s:

```shell
kubectl get ClusterDeployments -A
```

You'll see output something like this:

```console
NAMESPACE    NAME                        READY   STATUS
kcm-system   my-aws-clusterdeployment1   True    ClusterDeployment is ready
```

## Tear down the managed cluster

To tear down the managed cluster, delete the `ClusterDeployment`:

```shell
kubectl delete ClusterDeployment my-aws-clusterdeployment1 -n kcm-system
```

You'll see confirmation like this:

```console
clusterdeployment.k0rdent.mirantis.com "my-aws-clusterdeployment1" deleted
```

## Next Steps

Now that you've finished the k0rdent QuickStart, we have some suggestions for what to do next:

Check out the [Administrator Guide](admin-before.md) ...

* For a more detailed view of k0rdent setup for production
* For details about setting up k0rdent to manage clusters on VMware and OpenStack
* For details about using k0rdent with cloud Kubernetes distros: AWS EKS and Azure AKS

Or check out the [Demos Repository](https://github.com/k0rdent/demos) for fast, makefile-driven demos of k0rdent's key features!
