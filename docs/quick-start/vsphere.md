# vSphere Quick Start

## Prerequisites

1. `kubectl` CLI installed locally.
2. vSphere instance version `6.7.0` or higher.
3. vSphere account with appropriate [privileges](#vsphere-privileges).
4. [Image template](#image-template).
5. [vSphere network](#vsphere-network) with DHCP enabled.

### vSphere privileges

To function properly the user assigned to vSphere provider should be able to
manipulate vSphere resources. The following is the general overview of the
required privileges:

- `Virtual machine` - full permissions are required
- `Network` - `Assign network` is sufficient
- `Datastore` - it should be possible for user to manipulate virtual machine
  files and metadata

In addition to that specific CSI driver permissions are required see
[the official doc](https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/2.0/vmware-vsphere-csp-getting-started/GUID-0AB6E692-AA47-4B6A-8CEA-38B754E16567.html)
to get more information on CSI specific permissions.

### Image template

You can use pre-buit image templates from [CAPV project](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/main/README.md#kubernetes-versions-with-published-ovas)
or build your own.

When building your own image make sure that vmware tools and cloud-init are
installed and properly configured.

You can follow [official open-vm-tools guide](https://docs.vmware.com/en/VMware-Tools/11.0.0/com.vmware.vsphere.vmwaretools.doc/GUID-C48E1F14-240D-4DD1-8D4C-25B6EBE4BB0F.html)
on how to correctly install vmware-tools.

When setting up cloud-init you can refer to [official docs](https://cloudinit.readthedocs.io/en/latest/index.html)
and specifically [vmware datasource docs](https://cloudinit.readthedocs.io/en/latest/reference/datasources/vmware.html)
for extended information regarding cloud-init on vSphere.

### vSphere network

When creating network make sure that it has DHCP service.

Also make sure that the part of your network is out of DHCP range (e.g. network
172.16.0.0/24 with DHCP range 172.16.0.100-172.16.0.254). This is needed to make
sure that LB services will not create any IP conflicts in the network.

## Step 1: Create a Secret Object with the username and password

The Secret stores the username and password for your vSphere instance.

Save the Secret YAML into a file named `vsphere-cluster-identity-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vsphere-cluster-identity-secret
  namespace: hmc-system
stringData:
  username: <user>
  password: <Passw0rd>
type: Opaque
```

Apply the YAML to your cluster using the following command:

```bash
kubectl apply -f vsphere-cluster-identity-secret.yaml
```

## Step 2: Create the VSphereClusterIdentity Object

This object defines the credentials CAPV will use to manage vSphere resources.

Save the VSphereClusterIdentity YAML into a file named `vsphere-cluster-identity.yaml`:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: VSphereClusterIdentity
metadata:
  name: vsphere-cluster-identity
  namespace: hmc-system
spec:
  secretName: vsphere-cluster-identity-secret
  allowedNamespaces:
    selector:
      matchLabels: {}
```

Apply the YAML to your cluster:

```bash
  kubectl apply -f vsphere-cluster-identity.yaml
```

## Step 3: Create the 2A Credential Object

Create a YAML with the specificaion of our credential and safe it as `vsphere-cluster-identity-cred.yaml`

> The `kind:` must be `VSphereClusterIdentity` and the `name:` must match of the `VSphereClusterIdentity` object.

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: Credential
metadata:
  name: vsphere-cluster-identity-cred
  namespace: hmc-system
spec:
  identityRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: VSphereClusterIdentity
    name: vsphere-cluster-identity
    namespace: hmc-system
```

Apply the YAML to your cluster:

```bash
kubectl apply -f aws-cluster-identity-cred.yaml
```

## Step 4: Create the first managedCluster

Create a YAML with the specificaion of your managed Cluster and safe it as `my-vsphere-managedcluster1.yaml`

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: ManagedCluster
metadata:
  name: my-vsphere-managedcluster1
  namespace: hmc-system
spec:
  template: vsphere-standalone-cp-0-0-2
  credential: vsphere-cluster-identity-cred
  config:
    vsphere:
      server: <VSPHERE_SERVER>
      thumbprint: <VSPHERE_THUMBPRINT>
      datacenter: <VSPHERE_DATACENTER>
      datastore: <VSPHERE_DATASTORE>
      resourcePool: <VSPHERE_RESOURCEPOOL>
      folder: <VSPHERE_FOLDER>
      username: <VSPHERE_USER>
      password: <VSPHERE_PASSWORD>
    controlPlaneEndpointIP: <VSPHERE_CONTROL_PLANE_ENDPOINT>

    controlPlane:
      ssh:
        user: ubuntu
        publicKey: <VSPHERE_SSH_KEY>
      vmTemplate: <VSPHERE_VM_TEMPLATE>
      network: <VSPHERE_NETWORK>

    worker:
      ssh:
        user: ubuntu
        publicKey: <VSPHERE_SSH_KEY>
      vmTemplate: <VSPHERE_VM_TEMPLATE>
      network: <VSPHERE_NETWORK>
```

> For more information about the config options, see the [vSphere Template Parameters](../clustertemplates/vsphere/template-parameters.md).

Follow along the creation of the cluster

```bash
kubectl -n hmc-system get managedcluster.hmc.mirantis.com my-vsphere-managedcluster1  --watch
```

After the cluster is `Ready` you can access it via the kubeconfig, like this:

```bash
kubectl -n hmc-system get secret my-vsphere-managedcluster1-kubeconfig -o jsonpath='{.data.value}' | base64 -d > my-vsphere-managedcluster1-kubeconfig.kubeconfig
KUBECONFIG="my-vsphere-managedcluster1-kubeconfig.kubeconfig" kubectl get pods -A
```