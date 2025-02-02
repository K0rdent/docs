# Deploying a Hosted Control Plane

A hosted control plane is a Kubernetes setup in which the control plane components (such as the API server, 
etcd, and controllers) run inside the management cluster instead of separate controller nodes. This 
architecture centralizes control plane management, and improves scalability by sharing resources in the management cluster.
Hosted control planes are managed by [k0smotron](https://k0smotron.io/).

Instructions for setting up a hosted control plane vary slighting depending on the provider.

## AWS Hosted Control Plane Deployment

Follow these steps to set up a k0smotron-hosted control plane on AWS: 

1. Prerequisites

    Before proceeding, make sure you have the following:

    - A management Kubernetes cluster (Kubernetes v1.28 or later) deployed on AWS with [k0rdent installed](admin-installation.md).
    - A [default storage class](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/) configured on the management cluster to support Persistent Volumes.
    - The VPC ID where the worker nodes will be deployed.
    - The Subnet ID and Availability Zone (AZ) for the worker nodes.
    - The AMI ID for the worker nodes (Amazon Machine Image ID for the desired OS and Kubernetes version).

    > IMPORTANT:  
    > All control plane components for your hosted cluster will reside in the management cluster. The management cluster 
    > must have sufficient resources to handle these additional workloads.

2. Networking

    To deploy a hosted control plane, the necessary AWS networking resources must already exist or be created. If you're 
    using the same VPC and subnets as your management cluster, you can reuse these resources.

    If your management cluster was deployed using the Cluster API Provider AWS (CAPA), you can gather the required 
    networking details using the following commands:

    Retrieve the VPC ID:
    ```shell
    kubectl get awscluster <cluster-name> -o go-template='{{.spec.network.vpc.id}}'
    ```

    Retrieve Subnet ID:
    ```shell
    kubectl get awscluster <cluster-name> -o go-template='{{(index .spec.network.subnets 0).resourceID}}'
    ```

    Retrieve Availability Zone:
    ```shell
    kubectl get awscluster <cluster-name> -o go-template='{{(index .spec.network.subnets 0).availabilityZone}}'
    ```

    Retrieve Security Group:
    ```shell
    kubectl get awscluster <cluster-name> -o go-template='{{.status.networkStatus.securityGroups.node.id}}'
    ```

    Retrieve AMI ID:
    ```shell
    kubectl get awsmachinetemplate <cluster-name>-worker-mt -o go-template='{{.spec.template.spec.ami.id}}'
    ```

    > TIP:  
    > If you want to use different VPCs or regions for your management and hosted clusters, you’ll need to configure additional networking, such as [VPC peering](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/vpc-peering.html), to allow communication between them.


3. Create the ClusterDeployment manifest

    Once you've collected all the necessary data, you can create the `ClusterDeployment` manifest. This file tells k0rdent how to 
    deploy and manage the hosted control plane. Below is an example:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: aws-hosted-cp
    spec:
      template: aws-hosted-cp-0-0-3
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
    > The example above uses the `us-west-1` region, but you should use the region of your VPC.

### Generate the `ClusterDeployment` Manifest

    To simplify the creation of a `ClusterDeployment` manifest, you can use the following template, which dynamically 
    inserts the appropriate values:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: aws-hosted
    spec:
      template: aws-hosted-cp-0-0-3
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
        {{- end }}
        instanceType: t3.medium
        securityGroupIDs:
          - "{{.status.networkStatus.securityGroups.node.id}}"
    ```

    Save this template as `clusterdeployment.yaml.tpl`, then generate your manifest using the following command:

    ```shell
    kubectl get awscluster <cluster-name> -o go-template="$(cat clusterdeployment.yaml.tpl)" > clusterdeployment.yaml
    ```

4. Apply the `ClusterTemplate`

    Nothing actually happens until you apply the `ClusterDeployment` manifest to create a new cluster deployment:

    ```shell
    kubectl apply -f clusterdeployment.yaml -n kcm-system
    ```

### Deployment Tips

Here are some additional tips to help with deployment:

1. Controller and Template Availability:

   Make sure the kcm controller image and templates are available in a public or accessible repository.

2. Install Charts and Templates:

   If you're using a custom repository, run the following commands with the appropriate `kubeconfig`:

   ```shell
   KUBECONFIG=kubeconfig IMG="ghcr.io/k0rdent/kcm/controller-ci:v0.0.1-179-ga5bdf29" REGISTRY_REPO="oci://ghcr.io/k0rdent/kcm/charts-ci" make dev-apply
   KUBECONFIG=kubeconfig make dev-templates
   ```

3. Mark Infrastructure as Ready:

   To scale up the `MachineDeployment`, manually mark the infrastructure as ready:
   ```shell
   kubectl patch AWSCluster <hosted-cluster-name> --type=merge --subresource status --patch '{"status": {"ready": true}}' -n kcm-system
   ```
   For more details on why this is necessary, [click here](https://docs.k0smotron.io/stable/capi-aws/#:~:text=As%20we%20are%20using%20self%2Dmanaged%20infrastructure%20we%20need%20to%20manually%20mark%20the%20infrastructure%20ready.%20This%20can%20be%20accomplished%20using%20the%20following%20command).

## Azure Hosted Control Plane Deployment

Follow these steps to set up a k0smotron-hosted control plane on Azure:

1. Prerequisites

    Before you start, make sure you have the following:

    - A management Kubernetes cluster (Kubernetes v1.28+) deployed on Azure with [k0rdent installed](admin-installation.md).
    - A [default storage class](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/) configured 
      on the management cluster to support Persistent Volumes.

    > NOTE:  
    > All control plane components for managed clusters will run in the management cluster. Make sure the management cluster 
      has sufficient CPU, memory, and storage to handle the additional workload.

2.  Gather pre-existing resources

    In a hosted control plane setup, some Azure resources must exist before deployment and must be explicitly 
    provided in the `ClusterDeployment` configuration. These resources can also be reused by the management cluster.

    If you deployed your Azure Kubernetes cluster using the Cluster API Provider for Azure (CAPZ), you can retrieve 
    the required information using the following commands:

    Location:
    ```shell
    kubectl get azurecluster <cluster-name> -o go-template='{{.spec.location}}'
    ```

    Subscription ID:
    ```shell
    kubectl get azurecluster <cluster-name> -o go-template='{{.spec.subscriptionID}}'
    ```

    Resource Group:
    ```shell
    kubectl get azurecluster <cluster-name> -o go-template='{{.spec.resourceGroup}}'
    ```

    VNet Name:
    ```shell
    kubectl get azurecluster <cluster-name> -o go-template='{{.spec.networkSpec.vnet.name}}'
    ```

    Subnet Name:
    ```shell
    kubectl get azurecluster <cluster-name> -o go-template='{{(index .spec.networkSpec.subnets 1).name}}'
    ```

    Route Table Name:
    ```shell
    kubectl get azurecluster <cluster-name> -o go-template='{{(index .spec.networkSpec.subnets 1).routeTable.name}}'
    ```

    Security Group Name:
    ```shell
    kubectl get azurecluster <cluster-name> -o go-template='{{(index .spec.networkSpec.subnets 1).securityGroup.name}}'
    ```

3. Create the ClusterDeployment manifest

    After collecting the required data, create a `ClusterDeployment` manifest to configure the hosted control plane. It should
    look something like this:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: azure-hosted-cp
    spec:
      template: azure-hosted-cp-0-0-2
      credential: azure-credential
      config:
        clusterLabels: {}
        location: "westus"
        subscriptionID: ceb131c7-a917-439f-8e19-cd59fe247e03
        vmSize: Standard_A4_v2
        resourceGroup: mgmt-cluster
        network:
          vnetName: mgmt-cluster-vnet
          nodeSubnetName: mgmt-cluster-node-subnet
          routeTableName: mgmt-cluster-node-routetable
          securityGroupName: mgmt-cluster-node-nsg
    ```

### Generate the `ClusterDeployment` Manifest

    To simplify the creation of a `ClusterDeployment` manifest, you can use the following template, which dynamically inserts 
    the appropriate values:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: azure-hosted-cp
    spec:
      template: azure-hosted-cp-0-0-2
      credential: azure-credential
      config:
        clusterLabels: {}
        location: "{{.spec.location}}"
        subscriptionID: "{{.spec.subscriptionID}}"
        vmSize: Standard_A4_v2
        resourceGroup: "{{.spec.resourceGroup}}"
        network:
          vnetName: "{{.spec.networkSpec.vnet.name}}"
          nodeSubnetName: "{{(index .spec.networkSpec.subnets 1).name}}"
          routeTableName: "{{(index .spec.networkSpec.subnets 1).routeTable.name}}"
          securityGroupName: "{{(index .spec.networkSpec.subnets 1).securityGroup.name}}"
    ```

    Save this YAML as `clusterdeployment.yaml.tpl` and render the manifest with the following command:
    ```shell
    kubectl get azurecluster <management-cluster-name> -o go-template="$(cat clusterdeployment.yaml.tpl)" > clusterdeployment.yaml
    ```

4. Create the `ClusterDeployment`

    To actually create the cluster, apply the `ClusterDeployment` manifest to the management cluster, as in:

    ```shell
    kubectl apply clusterdeployment.yaml -n kcm-system
    ```
      
5. Manually update the `AzureCluster` object

    Due to a limitation in k0smotron, (see [k0sproject/k0smotron#668](https://github.com/k0sproject/k0smotron/issues/668)), 
    after applying the `ClusterDeployment` manifest, you must manually update the status of the `AzureCluster` object.

    Use the following command to set the `AzureCluster` object status to `Ready`:

    ```shell
    kubectl patch azurecluster <cluster-name> --type=merge --subresource status --patch '{"status": {"ready": true}}'
    ```

### Important Notes on Cluster Deletion

Due to these same k0smotron limitations, you **must** take some manual steps in order to delete a cluster properly:

1. Add a Custom Finalizer to the AzureCluster Object:

   To prevent the `AzureCluster` object from being deleted too early, add a custom finalizer:

   ```shell
   kubectl patch azurecluster <cluster-name> --type=merge --patch '{"metadata": {"finalizers": ["manual"]}}'
   ```

2. Delete the ClusterDeployment:
   After adding the finalizer, delete the `ClusterDeployment` object as usual. Confirm that all `AzureMachines` objects have been deleted successfully.

3. Remove Finalizers from Orphaned AzureMachines:

   If any `AzureMachines` are left orphaned, delete their finalizers manually after confirming no VMs remain in Azure. Use this command to remove the finalizer:

   ```shell
   kubectl patch azuremachine <machine-name> --type=merge --patch '{"metadata": {"finalizers": []}}'
   ```

4. Allowing Updates to Orphaned Objects:

    If Azure admission controls prevent updates to orphaned objects, you must disable the associated `MutatingWebhookConfiguration` by deleting it:

    ```shell
    kubectl delete mutatingwebhookconfiguration <webhook-name>
    ```

## vSphere Hosted Control Plane Deployment

Follow these steps to set up a k0smotron-hosted control plane on vSphere. 

1. Prerequisites

    Before you start, make sure you have the following:

    - A management Kubernetes cluster (Kubernetes v1.28+) deployed on vSphere with [k0rdent installed](admin-installation.md).

    All control plane components for managed clusters will reside in the management cluster. Make sure the management 
    cluster has sufficient resources (CPU, memory, and storage) to handle these workloads.

2. Create the `ClusterDeployment` Manifest

The `ClusterDeployment` manifest for vSphere-hosted control planes is similar to standalone control plane deployments. 
For a detailed list of parameters, refer to our discussion of [Template parameters for vSphere](template-vsphere.md).

> IMPORTANT: 
> The vSphere provider requires you to specify the control plane endpoint IP before deploying the cluster. This IP 
> address must match the one assigned to the k0smotron load balancer (LB) service.  
> Use an annotation supported by your load balancer provider to assign the control plane endpoint IP to the k0smotron 
> service. For example, the manifest below includes a `kube-vip` annotation.

`ClusterDeployment`s for vSphere-based clusters include a `.spec.config.vsphere` object that contains vSphere-specific
parameters. For example:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: cluster-1
spec:
  template: vsphere-hosted-cp-0-0-2
  credential: vsphere-credential
  config:
    clusterLabels: {}
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

For more information on these parameters, see the [Template reference for vsphere](template-vsphere.md). 