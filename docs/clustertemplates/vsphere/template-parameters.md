# vSphere cluster template parameters

## ClusterDeployment parameters

To create a cluster deployment a number of parameters should be passed to the
`ClusterDeployment` object.

### Parameter list

The following is the list of vSphere specific parameters, which are _required_
for successful cluster creation.

| Parameter                             | Example                               | Description                                                     |
|---------------------------------------|---------------------------------------|-----------------------------------------------------------------|
| `.spec.config.vsphere.server`         | `vcenter.example.com`                 | Address of the vSphere instance                                 |
| `.spec.config.vsphere.thumbprint`     | `"00:00:00:..."`                      | Certificate thumbprint                                          |
| `.spec.config.vsphere.datacenter`     | `DC`                                  | Datacenter name                                                 |
| `.spec.config.vsphere.datastore`      | `/DC/datastore/DS`                    | Datastore path                                                  |
| `.spec.config.vsphere.resourcePool`   | `/DC/host/vCluster/Resources/ResPool` | Resource pool path                                              |
| `.spec.config.vsphere.folder`         | `/DC/vm/example`                      | Folder path                                                     |
| `.spec.config.controlPlane.network`   | `/DC/network/vm_net`                  | Network path for `controlPlane`                                 |
| `.spec.config.worker.network`         | `/DC/network/vm_net`                  | Network path for `worker`                                       |
| `.spec.config.*.ssh.publicKey`        | `"ssh-ed25519 AAAA..."`               | SSH public key in `authorized_keys` format                      |
| `.spec.config.*.vmTemplate`           | `/DC/vm/templates/ubuntu`             | VM template image path                                          |
| `.spec.config.controlPlaneEndpointIP` | `172.16.0.10`                         | `kube-vip` vIP which will be created for control plane endpoint |

To obtain vSphere certificate thumbprint you can use the following command:

```bash
curl -sw %{certs} https://vcenter.example.com | openssl x509 -sha256 -fingerprint -noout | awk -F '=' '{print $2}'
```

[`govc`](https://github.com/vmware/govmomi/blob/main/govc/README.md), a vSphere CLI, can also help to discover proper values for some of the parameters:

```bash
# vsphere.datacenter
govc ls

# vsphere.datastore
govc ls /*/datastore/*

# vsphere.resourcePool
govc ls /*/host/*/Resources/*

# vsphere.folder
govc ls -l /*/vm/**

# controlPlane.network, worker.network
govc ls /*/network/*

# *.vmTemplate
govc vm.info -t '*'
```

> NOTE:
> Follow official `govc` installation instructions from [here](https://github.com/vmware/govmomi/blob/main/govc/README.md#installation),
> and `govc` usage guide is [here](https://github.com/vmware/govmomi/blob/main/govc/README.md#usage).
>
> Minimal `govc` configuration requires setting: `GOVC_URL`, `GOVC_USERNAME`, `GOVC_PASSWORD` environment variables.


## Example of ClusterDeployment CR

With all above parameters provided your `ClusterDeployment` can look like this:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: cluster-1
spec:
  template: vsphere-standalone-cp-0-0-2
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

    controlPlane:
      ssh:
        user: ubuntu
        publicKey: |
          ssh-rsa AAA...
      rootVolumeSize: 50
      cpus: 2
      memory: 4096
      vmTemplate: "/DC/vm/template"
      network: "/DC/network/Net"

    worker:
      ssh:
        user: ubuntu
        publicKey: |
          ssh-rsa AAA...
      rootVolumeSize: 50
      cpus: 2
      memory: 4096
      vmTemplate: "/DC/vm/template"
      network: "/DC/network/Net"
```


## SSH

Currently SSH configuration on vSphere expects that user is already created
during template creation. Because of that you must pass username along with SSH
public key to configure SSH access.


SSH public key can be passed to `.spec.config.ssh.publicKey` (in case of
hosted CP) parameter or `.spec.config.controlPlane.ssh.publicKey` and
`.spec.config.worker.ssh.publicKey` parameters (in case of standalone CP) of the
`ClusterDeployment` object.

SSH public key must be passed literally as a string.

Username can be passed to `.spec.config.controlPlane.ssh.user`,
`.spec.config.worker.ssh.user` or `.spec.config.ssh.user` depending on you
deployment model.

## VM resources

The following parameters are used to define VM resources:

| Parameter         | Example | Description                                                          |
|-------------------|---------|----------------------------------------------------------------------|
| `.rootVolumeSize` | `50`    | Root volume size in GB (can't be less than one defined in the image) |
| `.cpus`           | `2`     | Number of CPUs                                                       |
| `.memory`         | `4096`  | Memory size in MB                                                    |

The resource parameters are the same for hosted and standalone CP deployments,
but they are positioned differently in the spec, which means that they're going to:

- `.spec.config` in case of hosted CP deployment.
- `.spec.config.controlPlane` in in case of standalone CP for control plane
  nodes.
- `.spec.config.worker` in in case of standalone CP for worker nodes.

## VM Image and network

To provide image template path and network path the following parameters must be
used:

| Parameter     | Example           | Description         |
|---------------|-------------------|---------------------|
| `.vmTemplate` | `/DC/vm/template` | Image template path |
| `.network`    | `/DC/network/Net` | Network path        |

As with resource parameters the position of these parameters in the
`ClusterDeployment` depends on deployment type and these parameters are used in:

- `.spec.config` in case of hosted CP deployment.
- `.spec.config.controlPlane` in in case of standalone CP for control plane
  nodes.
- `.spec.config.worker` in in case of standalone CP for worker nodes.
