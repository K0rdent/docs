# AWS Quick Start

Much of the following includes the process of setting up credentials for AWS.
To better understand how k0rdent uses credentials, read the
[Credential system](../credential/main.md).

## Prerequisites

### k0rdent Management Cluster

You need a Kubernetes cluster with [kcm installed](installation.md).

### Software prerequisites

The AWS `clusterawsadm` tool is required to bootstrap an AWS Account. Install it
by following the [AWS clusterawsadm installation instructions](https://github.com/kubernetes-sigs/cluster-api-provider-aws?tab=readme-ov-file#clusterawsadm).

## EKS Deployment

- Additional EKS steps and verifications are described in [EKS clusters](../clustertemplates/aws/template-parameters.md#eks-templates).

### Configure AWS IAM

Before launching a cluster on AWS, you need to set up your AWS infrastructure
with the necessary IAM policies and service account.

> NOTE:
> 
> Skip steps below if you've already configured IAM policy for your AWS account

1. To use `clusterawsadm`, you must have an administrative user in an AWS
   account. Once you have that administrator user, set your environment
   variables:

    ```bash
    export AWS_REGION=<aws-region>
    export AWS_ACCESS_KEY_ID=<admin-user-access-key>
    export AWS_SECRET_ACCESS_KEY=<admin-user-secret-access-key>
    export AWS_SESSION_TOKEN=<session-token> # Optional. If you are using Multi-Factor Auth.
    ```

2. After these are set, run this command to create the IAM CloudFormation stack:

    ```bash
    clusterawsadm bootstrap iam create-cloudformation-stack
    ```

## Step 1: Create AWS IAM User

1. Create an AWS IAM user with the following policies assigned:

    - `control-plane.cluster-api-provider-aws.sigs.k8s.io`
    - `controllers.cluster-api-provider-aws.sigs.k8s.io`
    - `nodes.cluster-api-provider-aws.sigs.k8s.io`

2. Create Access Key for the IAM user.

    In the AWS IAM Console, create the Access Key for the IAM user and download
    its items (ID and Secret).

    You should have an `AccessKeyID` and a `SecretAccessKey` that look like the
    following:

    | Access key ID      | Secret access key   |
    |--------------------|---------------------|
    | AKIAQF+EXAMPLE     | EdJfFar6+example    |

## Step 2: Create the IAM Credentials Secret on k0rdent Management Cluster

Save the `Secret` YAML to a file named `aws-cluster-identity-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-cluster-identity-secret
  namespace: kcm-system
type: Opaque
stringData:
  AccessKeyID: AKIAQF+EXAMPLE
  SecretAccessKey: EdJfFar6+example
```

Apply the YAML to your cluster using the following command:

```bash
kubectl apply -f aws-cluster-identity-secret.yaml
```

> WARNING:
> 
> The secret must be created in the same `Namespace` where the CAPA provider is
> running. In case of k0rdent it's currently `kcm-system`. Placing secret in
> any other `Namespace` will result in the controller not able to read it.

## Step 3: Create AWSClusterStaticIdentity Object

Save the `AWSClusterStaticIdentity` YAML into a file named
`aws-cluster-identity.yaml`:

> NOTE:
> 
> `.spec.secretRef` must match `.metadata.name` of the secret that was created in
> the previous step.

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSClusterStaticIdentity
metadata:
  name: aws-cluster-identity
spec:
  secretRef: aws-cluster-identity-secret
  allowedNamespaces:
    selector:
      matchLabels: {}
```

Apply the YAML to your cluster:

```bash
kubectl apply -f aws-cluster-identity.yaml
```

## Step 4: Create the kcm Credential Object

Create a YAML with the specification of your credential and save it as
`aws-cluster-identity-cred.yaml`.

> NOTE:
> 
> `.spec.identityRef.kind` must be `AWSClusterStaticIdentity` and the
> `.spec.identityRef.name` must match the `.metadata.name` of the
> `AWSClusterStaticIdentity` object.

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

Apply the YAML to your cluster:

```bash
kubectl apply -f aws-cluster-identity-cred.yaml
```

## Step 5: Create Your First Cluster Deployment

Create a YAML with the specification of your Cluster Deployment and save it as
`my-aws-clusterdeployment1.yaml`.

Here is an example of a `ClusterDeployment` YAML file:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-aws-clusterdeployment1
  namespace: kcm-system
spec:
  template: aws-standalone-cp-0-0-4
  credential: aws-cluster-identity-cred
  config:
    region: us-west-2
    controlPlane:
      instanceType: t3.small
    worker:
      instanceType: t3.small
```

Apply the YAML to your management cluster:

```bash
kubectl apply -f my-aws-clusterdeployment1.yaml
```

There will be a delay as the cluster finishes provisioning. Follow the
provisioning process with the following command:

```bash
kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com my-aws-clusterdeployment1 --watch
```

After the cluster is `Ready`, you can access it via the kubeconfig, like this:

```bash
kubectl -n kcm-system get secret my-aws-clusterdeployment1-kubeconfig -o jsonpath='{.data.value}' | base64 -d > my-aws-clusterdeployment1-kubeconfig.kubeconfig
```

```bash
KUBECONFIG="my-aws-clusterdeployment1-kubeconfig.kubeconfig" kubectl get pods -A
```
