# AWS template parameters

## AWS AMI

By default AMI ID will be looked up automatically using the latest Amazon Linux 2 image.

You can override lookup parameters to search your desired image automatically or
use AMI ID directly.
If both AMI ID and lookup parameters are defined AMI ID will have higher precedence.

### Image lookup

To configure automatic AMI lookup 3 parameters are used:

- `.imageLookup.format` - used directly as value for the `name` filter
(see the [describe-images filters](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html#describe-images)).
  - Supports substitutions for `{{.BaseOS}}` and `{{.K8sVersion}}` with the base OS
and kubernetes version, respectively.

- `.imageLookup.org` - AWS org ID which will be used as value for the `owner-id`
filter.

- `.imageLookup.baseOS` - will be used as value for `{{.BaseOS}}` substitution in
the `.imageLookup.format` string.

### AMI ID

AMI ID can be directly used in the `.amiID` parameter.

#### CAPA prebuilt AMIs

Use `clusterawsadm` to get available AMIs to deploy cluster deployment:

```bash
clusterawsadm ami list
```

For details, see [Pre-built Kubernetes AMIs](https://cluster-api-aws.sigs.k8s.io/topics/images/built-amis.html).

## SSH access to cluster nodes

To access the nodes using the SSH protocol, several things should be configured:

- An SSH key added in the region where you want to deploy the cluster
- Bastion host is enabled

### SSH keys

Only one SSH key is supported and it should be added in AWS prior to creating
the `ClusterDeployment` object. The name of the key should then be placed under `.spec.config.sshKeyName`.

The same SSH key will be used for all machines and a bastion host.

To enable bastion you should add `.spec.config.bastion.enabled` option in the
`ClusterDeployment` object to `true`.

Full list of the bastion configuration options could be fould in [CAPA docs](https://cluster-api-aws.sigs.k8s.io/crd/#infrastructure.cluster.x-k8s.io/v1beta1.Bastion).

The resulting `ClusterDeployment` can look like this:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: cluster-1
spec:
  template: aws-standalone-cp-0-0-5
  credential: aws-cred
  config:
    sshKeyName: foobar
    bastion:
      enabled: true
...
```

## EKS templates

> WARNING:
> When deploying EKS cluster please note that
> [additional steps](vpc-removal.md) may be needed for proper VPC removal.

EKS templates use the parameters similar to AWS and resulting EKS
`ClusterDeployment` can look like this:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: cluster-1
spec:
  template: aws-eks-0-0-3
  credential: aws-cred
  config:
    sshKeyName: foobar
    region: ${AWS_REGION}
    workersNumber: 1
...
```
