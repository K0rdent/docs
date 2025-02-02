# AWS template parameters

## AWS AMI

By default k0rdent looks up the AMI ID automatically, using the latest Amazon Linux 2 image.

You can override lookup parameters to search your desired image automatically or you can
use a specific AMI ID directly.
If both the AMI ID and lookup parameters are defined, the AMI ID will have higher precedence.

### Image lookup

To configure automatic AMI lookup, k0rdent uses three parameters:

- `.imageLookup.format` - Used directly as a value for the `name` filter
(see the [describe-images filters](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html#describe-images)).
This field supports substitutions for `{{.BaseOS}}` and `{{.K8sVersion}}` with the base OS
and kubernetes version, respectively.

- `.imageLookup.org` - The AWS org ID that will be used as value for the `owner-id`
filter.

- `.imageLookup.baseOS` - The string to be used as a value for the `{{.BaseOS}}` substitution in
the `.imageLookup.format` string.

### AMI ID

The AMI ID can be directly used in the `.amiID` parameter.

#### CAPA prebuilt AMIs

Use `clusterawsadm` to get available AMIs to create a `ClusterDeployment`:

```bash
clusterawsadm ami list
```

For details, see [Pre-built Kubernetes AMIs](https://cluster-api-aws.sigs.k8s.io/topics/images/built-amis.html).

## SSH access to cluster nodes

To access nodes using SSH you'll need to do two things:

- Add an SSH key added in the region where you want to deploy the cluster
- Enable Bastion host is enabled

### SSH keys

Only one SSH key is supported and it should be added in AWS prior to creating
the `ClusterDeployment` object. The name of the key should then be placed under the `.spec.config.sshKeyName`.

The same SSH key will be used for all machines and a bastion host.

To enable the bastion, set the `.spec.config.bastion.enabled` option in the
`ClusterDeployment` object to `true`.

You can get a full list of the bastion configuration options in the [CAPA docs](https://cluster-api-aws.sigs.k8s.io/crd/#infrastructure.cluster.x-k8s.io/v1beta1.Bastion).

The resulting `ClusterDeployment` might look something like this:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: cluster-1
spec:
  template: aws-standalone-cp-0-0-5
  credential: aws-cred
  config:
    clusterLabels: {}
    sshKeyName: foobar
    bastion:
      enabled: true
...
```

## EKS templates

> WARNING:
> When deploying EKS cluster please note that
> [additional steps](admin-troubleshooting-aws-vpcs.md) may be needed for proper VPC removal.

> WARNING:
> You may encounter an issue where EKS machines are not created due to the `ControlPlaneIsStable` preflight check
> failure during EKS cluster deployment. Please follow the
> [instruction](known-issues-eks.md#eks-machines-are-not-created-controlplaneisstable-preflight-check-failed)
> to apply the workaround.

> WARNING:
> You may encounter an issue where EKS machines are not created due to the `ControlPlaneIsStable` preflight check
> failure during EKS cluster deployment. Please follow the
> [instruction](known-issues-eks.md#eks-machines-are-not-created-controlplaneisstable-preflight-check-failed)
> to apply the workaround.

EKS templates use parameters similar to AWS and the resulting EKS
`ClusterDeployment` looks something like this:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: cluster-1
spec:
  template: aws-eks-0-0-3
  credential: aws-cred
  config:
    clusterLabels: {}
    sshKeyName: foobar
    region: ${AWS_REGION}
    workersNumber: 1
...
```

# AWS Hosted control plane deployment

This section covers setting up for a k0smotron hosted control plane on AWS.

## Prerequisites

Before starting you must have:

-   A management Kubernetes cluster (v1.28+) deployed on AWS with kcm installed on it
-   A default default storage class configured on the management cluster
-   A VPC ID for the worker nodes
-   A Subnet ID which will be used along with AZ information
-   An AMI ID which will be used to deploy worker nodes

Keep in mind that all control plane components for all cluster deployments will
reside in the management cluster.

## Networking

The networking resources in AWS which are needed for a cluster deployment can be
reused with a management cluster.

If you deployed your AWS Kubernetes cluster using Cluster API Provider AWS (CAPA)
you can obtain all the necessary data with the commands below or use the
template found below in the
[kcm ClusterDeployment manifest generation](#kcm-clusterdeployment-manifest-generation)
section.

If using the `aws-standalone-cp` template to deploy a hosted cluster it is
recommended to use a `t3.large` or larger instance type as the `kcm-controller`
and other provider controllers will need a large amount of resources to run.

**VPC ID**

```bash
    kubectl get awscluster <cluster-name> -o go-template='{{.spec.network.vpc.id}}'
```

**Subnet ID**

```bash
    kubectl get awscluster <cluster-name> -o go-template='{{(index .spec.network.subnets 0).resourceID}}'
```

**Availability zone**

```bash
    kubectl get awscluster <cluster-name> -o go-template='{{(index .spec.network.subnets 0).availabilityZone}}'
```

**Security group**
```bash
    kubectl get awscluster <cluster-name> -o go-template='{{.status.networkStatus.securityGroups.node.id}}'
```

**AMI id**

```bash
    kubectl get awsmachinetemplate <cluster-name>-worker-mt -o go-template='{{.spec.template.spec.ami.id}}'
```

If you want to use different VPCs/regions for your management or managed
clusters you should setup additional connectivity rules like
[VPC peering](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/vpc-peering.html).


## kcm ClusterDeployment manifest

With all the collected data your `ClusterDeployment` manifest will look similar to this:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: aws-hosted-cp
spec:
  template: aws-hosted-cp-0-0-4
  credential: aws-credential
  config:
    clusterLabels: {}
    vpcID: vpc-0a000000000000000
    region: us-west-1
    publicIP: true
    subnets:
      - id: subnet-0aaaaaaaaaaaaaaaa
        availabilityZone: us-west-1b
        isPublic: true
        natGatewayID: xxxxxx
        routeTableId: xxxxxx
      - id: subnet-1aaaaaaaaaaaaaaaa
        availabilityZone: us-west-1b
        isPublic: false
        routeTableId: xxxxxx
    instanceType: t3.medium
    securityGroupIDs:
      - sg-0e000000000000000
```

> NOTE:
> In this example we're using the `us-west-1` region, but you should use the
> region of your VPC.

## kcm ClusterDeployment manifest generation

Grab the following `ClusterDeployment` manifest template and save it to a file
named `clusterdeployment.yaml.tpl`:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: aws-hosted
spec:
  template: aws-hosted-cp-0-0-4
  credential: aws-credential
  config:
    clusterLabels: {}
    vpcID: "{{.spec.network.vpc.id}}"
    region: "{{.spec.region}}"
    subnets:
    {{- range $subnet := .spec.network.subnets }}
      - id: "{{ $subnet.resourceID }}"
        availabilityZone: "{{ $subnet.availabilityZone }}"
        isPublic: {{ $subnet.isPublic }}
        {{- if $subnet.isPublic }}
        natGatewayId: "{{ $subnet.natGatewayId }}"
        {{- end }}
        routeTableId: "{{ $subnet.routeTableId }}"
        zoneType: "{{ $subnet.zoneType }}"
    {{- end }}
    instanceType: t3.medium
    securityGroupIDs:
      - "{{.status.networkStatus.securityGroups.node.id}}"
```

Then run the following command to create the `clusterdeployment.yaml`:

```
kubectl get awscluster cluster -o go-template="$(cat clusterdeployment.yaml.tpl)" > clusterdeployment.yaml
```
## Deployment Tips
* Ensure kcm templates and the controller image are somewhere public and
  fetchable.
* For installing the kcm charts and templates from a custom repository, load
  the `kubeconfig` from the cluster and run the commands:

```
KUBECONFIG=kubeconfig IMG="ghcr.io/k0rdent/kcm/controller-ci:v0.0.1-179-ga5bdf29" REGISTRY_REPO="oci://ghcr.io/k0rdent/kcm/charts-ci" make dev-apply
KUBECONFIG=kubeconfig make dev-templates
```
* The infrastructure will need to manually be marked `Ready` to get the
  `MachineDeployment` to scale up.  You can patch the `AWSCluster` kind using
  the command:

```
KUBECONFIG=kubeconfig kubectl patch AWSCluster <hosted-cluster-name> --type=merge --subresource status --patch 'status: {ready: true}' -n kcm-system
```

For additional information on why this is required [click here](https://docs.k0smotron.io/stable/capi-aws/#:~:text=As%20we%20are%20using%20self%2Dmanaged%20infrastructure%20we%20need%20to%20manually%20mark%20the%20infrastructure%20ready.%20This%20can%20be%20accomplished%20using%20the%20following%20command).

