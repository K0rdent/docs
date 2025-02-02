# OpenStack Machine parameters

## ClusterDeployment Parameters

To deploy an OpenStack cluster, the following are the primary parameters in the `ClusterDeployment` resource:

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Example</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>.spec.credential</td>
      <td><code>openstack-cluster-identity-cred</code></td>
      <td>Reference to the Credential object.</td>
    </tr>
    <tr>
      <td>.spec.template</td>
      <td><code>openstack-standalone-cp-0-0-1</code></td>
      <td>Reference to the ClusterTemplate.</td>
    </tr>
    <tr>
      <td>.spec.config.authURL</td>
      <td><code>https://keystone.yourorg.net/</code></td>
      <td>Keystone authentication endpoint for OpenStack.</td>
    </tr>
    <tr>
      <td>.spec.config.controlPlaneNumber</td>
      <td><code>3</code></td>
      <td>Number of control plane nodes.</td>
    </tr>
    <tr>
      <td>.spec.config.workersNumber</td>
      <td><code>2</code></td>
      <td>Number of worker nodes.</td>
    </tr>
    <tr>
      <td>.spec.config.clusterLabels<br>(optional)</td>
      <td><code>k0rdent: demo</code></td>
      <td>Labels to apply to the cluster. Used by <br>MultiClusterService.</td>
    </tr>
  </tbody>
</table>                  |

### SSH Configuration

`sshPublicKey` is the reference name for an existing SSH key configured in OpenStack.

- **ClusterDeployment**: Specify the SSH public key using the `.spec.config.controlPlane.sshPublicKey` and `.spec.config.worker.sshPublicKey` parameters (for the standlone control plane).

### Machine Configuration

Configurations for control plane and worker nodes are specified separately under `.spec.config.controlPlane` and `.spec.config.worker`:

| Parameter                  | Example                | Description                        |
|----------------------------|------------------------|------------------------------------|
| `flavor`                   | `m1.medium`           | OpenStack flavor for the instance.|
| `image.filter.name`        | `ubuntu-22.04-x86_64` | Name of the image.                |
| `sshPublicKey`             | `ramesses-pk`         | Reference name for an existing SSH key.|
| `securityGroups.filter.name`| `default`             | Security group for the instance.  |

> [!NOTE]
> Make sure `.spec.credential` references the `Credential` object.
> The recommended minimum vCPU value for the control plane flavor is 2, while for the worker node flavor, it is 1. For detailed information, refer to the [machine-flavor CAPI docs](https://github.com/kubernetes-sigs/cluster-api-provider-openstack/blob/main/docs/book/src/clusteropenstack/configuration.md#machine-flavor).

### Example ClusterDeployment

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-openstack-cluster-deployment
  namespace: kcm-system
spec:
  template: openstack-standalone-cp-0-0-2
  credential: openstack-cluster-identity-cred
  config:
    clusterLabels: {}
    clusterLabels:
      k0rdent: demo
    controlPlaneNumber: 1
    workersNumber: 1
    controlPlane:
      sshPublicKey: my-public-key
      flavor: m1.medium
      image:
        filter:
          name: ubuntu-22.04-x86_64
    worker:
      sshPublicKey: my-public-key
      flavor: m1.medium
      image:
        filter:
          name: ubuntu-22.04-x86_64
    authURL: https://my-keystone-openstack-url.com
    identityRef:
      name: openstack-cloud-config
      cloudName: openstack
      region: RegionOne
```