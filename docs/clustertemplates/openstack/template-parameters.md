# OpenStack Machine parameters

## ClusterDeployment Parameters

To deploy an OpenStack cluster, the following parameters are critical in the `ClusterDeployment` resource:

| Parameter                     | Example                               | Description                                         |
|-------------------------------|---------------------------------------|-----------------------------------------------------|
| `.spec.credential`            | `openstack-cluster-identity-cred`     | Reference to the Credential object.                 |
| `.spec.template`              | `openstack-standalone-cp-0-0-1`       | Reference to the ClusterTemplate.                   |
| `.spec.config.authURL`        | `https://keystone.yourorg.net/`       | Keystone authentication endpoint for OpenStack.     |
| `.spec.config.controlPlaneNumber` | `3`                               | Number of control plane nodes.                      |
| `.spec.config.workersNumber`  | `2`                                   | Number of worker nodes.                             |

### SSH Configuration

`sshPublicKey` is a reference name for an existing SSH key configured in OpenStack.

- **ClusterDeployment**: Specify the SSH public key using `.spec.config.controlPlane.sshPublicKey` and `.spec.config.worker.sshPublicKey` parameters (in case of standalone CP).

### Machine Configuration

Configurations for control plane and worker nodes are specified separately under `.spec.config.controlPlane` and `.spec.config.worker`:

| Parameter                  | Example                | Description                        |
|----------------------------|------------------------|------------------------------------|
| `flavor`                   | `m1.medium`           | OpenStack flavor for the instance.|
| `image.filter.name`        | `ubuntu-22.04-x86_64` | Name of the image.                |
| `sshPublicKey`             | `ramesses-pk`         | Reference name for an existing SSH key.|
| `securityGroups.filter.name`| `default`             | Security group for the instance.  |

> [!NOTE]
> Ensure `.spec.credential` references the `Credential` object (see [OpenStack-QuickStart](../../quick-start/openstack.md))
> The recommended minimum vCPU value for the control plane flavor is 2, while for the worker node flavor, it is 1. For detailed information, refer to the [machine-flavor CAPI docs](https://github.com/kubernetes-sigs/cluster-api-provider-openstack/blob/main/docs/book/src/clusteropenstack/configuration.md#machine-flavor).

### Example ClusterDeployment

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-openstack-cluster-deployment
  namespace: kcm-system
spec:
  template: openstack-standalone-cp-0-0-1
  credential: openstack-cluster-identity-cred
  config:
    controlPlaneNumber: 1
    workersNumber: 1
    controlPlane:
      sshPublicKey: ramesses-pk
      flavor: m1.medium
      image:
        filter:
          name: ubuntu-22.04-x86_64
    worker:
      sshPublicKey: ramesses-pk
      flavor: m1.medium
      image:
        filter:
          name: ubuntu-22.04-x86_64
    authURL: <OS_AUTH_URL>
```
