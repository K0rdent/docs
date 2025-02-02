# Creating and lifecycle-managing managed clusters

Once you've installed k0rdent, you can use it to create, manage, update and even upgrade clusters.

## Deploying a Cluster

k0rdent is designed to simplify the process of deploying and managing Kubernetes clusters across various cloud platforms. It does this through the use of `ClusterDeployment`s, which include all of the information k0rdent needs to know in order to create the cluster you're looking for. This `ClusterDeployment` system relies on predefined templates and credentials. 

A cluster deployment typically involves:

1. Setting up credentials for the infrastructure provider (for example, AWS, vSphere).
2. Choosing a template that defines the desired cluster configuration (for example, number of nodes, instance types).
3. Submitting the configuration for deployment and monitoring the process.

Follow these steps to deploy a standalone Kubernetes cluster tailored to your specific needs:

1. Create the `Credential` object

    Credentials are essential for k0rdent to communicate with the infrastructure provider (for example, AWS, Azure, vSphere). These credentials enable k0rdent to provision resources such as virtual machines, networking components, and storage.

    `Credential` objects are generally created ahead of time and made available to users, so before you look into creating a
    new one be sure what you're looking for doesn't already exist. You can see all of the existing `Credential` objects by 
    querying the management cluster:

    ```shell
    kubectl get credentials -n A
    ```

    If the `Credential` you need doesn't yet exist, go ahead and create it.

    Start by creating a `Credential` object that includes all required authentication details for your chosen infrastructure provider. Follow the instructions in the [chapter about credential management](admin-credentials.md), as well as the specific instructions for your [target infrastructure](admin-prepare.md).

    > TIP: 
    > Double-check to make sure that your credentials have sufficient permissions to create resources on the target infrastructure.

2. Select a Template

    Templates in k0rdent are predefined configurations that describe how the cluster should be set up. Templates include details such as:

    - The number and type of control plane and worker nodes.
    - Networking settings.
    - Regional deployment preferences.

    Templates act as a blueprint for creating a cluster. To see the list of available templates, use the following command:

    ```shell
    kubectl get clustertemplate -n kcm-system
    ```
    ```console
    NAMESPACE    NAME                            VALID
    kcm-system   adopted-cluster-0-0-2           true
    kcm-system   aws-eks-0-0-3                   true
    kcm-system   aws-hosted-cp-0-0-4             true
    kcm-system   aws-standalone-cp-0-0-5         true
    kcm-system   azure-aks-0-0-2                 true
    kcm-system   azure-hosted-cp-0-0-4           true
    kcm-system   azure-standalone-cp-0-0-5       true
    kcm-system   openstack-standalone-cp-0-0-2   true
    kcm-system   vsphere-hosted-cp-0-0-5         true
    kcm-system   vsphere-standalone-cp-0-0-5     true
    ```

    You can then get information on the actual template by describing it, as in:

    ```shell
    kubectl describe clustertemplate aws-standalone-cp-0-0-5 -n ksm-system
    ```

3. Create a ClusterDeployment YAML Configuration

    The `ClusterDeployment` object is the main configuration file that defines your cluster's specifications. It includes:

    - The template to use.
    - The credentials for the infrastructure provider.
    - Optional customizations such as instance types, regions, and networking.

    Create a `ClusterDeployment` configuration in a YAML file, following this structure:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: <cluster-name>
      namespace: <kcm-system-namespace>
    spec:
      template: <template-name>
      credential: <infrastructure-provider-credential-name>
      dryRun: <"true" or "false" (default: "false")>
      config:
        <cluster-configuration>
    ```

    You will of course want to replace the placeholders with actual values. (For more information about `dryRun` see [Understanding the Dry Run](appendix-dryrun.md)) For example, this is a simple AWS infrastructure provider `ClusterDeployment`:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: my-managed-cluster
      namespace: kcm-system
    spec:
      template: aws-standalone-cp-0-0-3
      credential: aws-credential
      dryRun: false
      config:
        clusterLabels: {}
        region: us-west-2
        controlPlane:
          instanceType: t3.small
        worker:
          instanceType: t3.small
    ```
    Note that the `.spec.credential` value should match the `.metadata.name` value of a created `Credential` object.

4. Apply the Configuration

    Once the `ClusterDeployment` configuration is ready, apply it to the k0rdent management cluster:

    ```shell
    kubectl apply -f clusterdeployment.yaml
    ```

    This step submits your deployment request to k0rdent. If you've set `dryRun` to `true` you can observe what would happen. Otherwise, k0rdent will go ahead and begin provisioning the necessary infrastructure.

5. Verify Deployment Status

    After submitting the configuration, verify that the `ClusterDeployment` object has been created successfully:

    ```shell
    kubectl -n <namespace> get clusterdeployment.kcm <cluster-name> -o=yaml
    ```

    The output shows the current status and any errors.

6. Monitor Provisioning

    k0rdent will now start provisioning resources (e.g., VMs, networks) and setting up the cluster. To monitor this process, run:

    ```shell
    kubectl -n <namespace> get cluster <cluster-name> -o=yaml
    ```

    > TIP:  
    > For a detailed view of the provisioning process, use the `clusterctl describe` command (note that this requires the [`clusterctl`](https://github.com/kubernetes-sigs/cluster-api/releases) CLI):

    ```shell
    clusterctl describe cluster <cluster-name> -n <namespace> --show-conditions all
    ```

7. Retrieve the Kubernetes Configuration

    When provisioning is complete, retrieve the kubeconfig file for the new cluster. This file enables you to interact with the cluster using `kubectl`:

    ```shell
    kubectl get secret -n <namespace> <cluster-name>-kubeconfig -o=jsonpath={.data.value} | base64 -d > kubeconfig
    ```
    You can then use this file to access the cluster, as in:

    ```shell
    export KUBECONFIG=kubeconfig
    kubectl get pods -A
    ```

    Store the kubeconfig file securely, as it contains authentication details for accessing the cluster.

## Updating a Single Standalone Cluster

k0rdent `ClusterTemplate`s are immutable, so the only way to change a `ClusterDeployment` is to change the template
that forms its basis. 

To update the `ClusterDeployment`, modify the `.spec.template` field to use the name of the new `ClusterTemplate`. 
This enables you to apply changes to the cluster configuration. These changes will then be applied to the actual 
cluster. For example, if the cluster currently uses `t2.large` instances, that will be specified in its current template. 
To change the cluster to use `t2.xlarge` instances, you would simply apply a template that references that new size; 
k0rdent will then realize the cluster is out of sync and will attempt to remedy the situation by updating the cluster.

Follow these steps to update the `ClusterDeployment`:

  1. Patch the `ClusterDeployment` with the new template

    Run the following command, replacing the placeholders with the appropriate values:

    ```shell
    kubectl patch clusterdeployment.kcm <cluster-name> -n <namespace> --patch '{"spec":{"template":"<new-template-name>"}}' --type=merge
    ```

  2. Check the status of the `ClusterDeployment`

    After applying the patch, verify the status of the `ClusterDeployment` object:

    ```shell
    kubectl get clusterdeployment.kcm <cluster-name> -n <namespace>
    ```

  3. Inspect the detailed status

    For more details, use the `-o=yaml` option to check the `.status.conditions` field:

    ```shell
    kubectl get clusterdeployment.kcm <cluster-name> -n <namespace> -o=yaml
    ```

Note that not all updates are possible; `ClusterTemplateChain` objects limit what templates can be applied.  Consider,
for example, this `ClusterTemplateChain`:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterTemplateChain
metadata:
  name: aws-standalone-cp-0.0.2
  namespace: kcm-system
spec:
  supportedTemplates:
    - name: aws-standalone-cp-0.0.1
      availableUpgrades:
        - name: aws-standalone-cp-0.0.2
    - name: aws-standalone-cp-0.0.2
```

As you can see from the `.spec`, the `aws-standalone-co-0.0.2` template can be applied to a cluster that also uses
the `aws-standalone-co-0.0.2` template, or it can be used as an upgrade from a cluster that uses `aws-standalone-co-0.0.1`.
You wouldn't be able to use this template to update a cluster that uses any other `ClusterTemplate`.

Similarly, the `AccessManagement` object must have properly configured `spec.accessRules` with a list of allowed 
`ClusterTemplateChain` object names and their namespaces. For more information, see [Template Life Cycle Management](template-intro.md#template-life-cycle-management).

> NOTE:  
> Support for displaying all available `ClusterTemplates` for updates in the `ClusterDeployment` status is planned.

<!-- TODO
## Scaling a Cluster 
## Upgrading a Single Standalone Cluster 
-->

## Cleanup

Especially when you are paying for cloud resources, it's crucial to clean up when you're finished.
Fortunately, k0rdent makes that straightforward.

Because a Kubernetes cluster is represented by a `ClusterDeployment`, when you delete that `ClusterDeployment`,
k0rdent deletes the cluster. For example:

```shell
kubectl delete clusterdeployment my-cluster-deployment -n kmc-system
```
```console
ClusterDeployment my-cluster-deployment deleted.
```

It takes time to delete these resources.