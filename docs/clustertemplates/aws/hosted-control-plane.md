# AWS Hosted control plane deployment

This section covers setting up for a k0smotron hosted control plane on AWS.

## Prerequisites

-   Management Kubernetes cluster (v1.28+) deployed on AWS with kcm installed on it
-   Default storage class configured on the management cluster
-   VPC ID for the worker nodes
-   Subnet ID which will be used along with AZ information
-   AMI ID which will be used to deploy worker nodes

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
  template: aws-hosted-cp-0-0-3
  credential: aws-credential
  config:
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
  template: aws-hosted-cp-0-0-3
  credential: aws-credential
  config:
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
