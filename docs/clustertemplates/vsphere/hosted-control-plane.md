# Hosted control plane (k0smotron) deployment

## Prerequisites

- Management Kubernetes cluster (v1.28+) deployed on vSphere with kcm installed
  on it

Keep in mind that all control plane components for all managed clusters will
reside in the management cluster.


## ClusterDeployment manifest

Hosted CP template has mostly identical parameters with standalone CP, you can
check them in the [template parameters](template-parameters.md) section.

> NOTE: **Important Note on Control Plane Endpoint IP Address**
> The vSphere provider requires the control plane endpoint IP to be specified
> before deploying the cluster. Ensure that this IP matches the IP assigned to
> the k0smotron load balancer (LB) service. Provide the control plane endpoint
> IP to the k0smotron service via an annotation accepted by your LB provider
> (e.g., the `kube-vip` annotation in the example below).

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: cluster-1
spec:
  template: vsphere-hosted-cp-0-0-2
  credential: vsphere-credential
  config:
    vsphere:
      server: vcenter.example.com
      thumbprint: "00:00:00"
      datacenter: "DC"
      datastore: "/DC/datastore/DC"
      resourcePool: "/DC/host/vCluster/Resources/ResPool"
      folder: "/DC/vm/example"
    controlPlaneEndpointIP: "172.16.0.10"
    ssh:
      user: ubuntu
      publicKey: |
        ssh-rsa AAA...
    rootVolumeSize: 50
    cpus: 2
    memory: 4096
    vmTemplate: "/DC/vm/template"
    network: "/DC/network/Net"
    k0smotron:
      service:
        annotations:
          kube-vip.io/loadbalancerIPs: "172.16.0.10"
```
