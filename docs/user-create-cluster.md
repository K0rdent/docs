# Deploying a Cluster

k0rdent simplifies the process of deploying and managing Kubernetes clusters across various cloud platforms through the use of `ClusterDeployment`s, which include all of the information k0rdent needs to know in order to create the cluster you want. This `ClusterDeployment` system relies on predefined templates and credentials. 

A cluster deployment typically involves:

1. Credentials for the infrastructure provider (for example, AWS, vSphere).
2. A template that defines the desired cluster configuration (for example, number of nodes, instance types).
3. Submitting the configuration for deployment and monitoring the process.

Follow these steps to deploy a standalone Kubernetes cluster:

1. Obtain the `Credential` object

    k0rdent needs credentials to communicate with the infrastructure provider (for example, AWS, Azure, vSphere). These credentials enable k0rdent to provision resources such as virtual machines, networking components, and storage.

    `Credential` objects are generally created ahead of time and made available to users. You can see all of the existing `Credential` objects by querying the management cluster:

    ```shell
    kubectl get credentials -n accounting
    ```
    When you find a `Credential` that looks appropriate, you can get more information by `describe`ing it, as in:

    ```shell
    kubectl describe credential accounting-cluster-credential -n accounting
    ```

    You'll see the YAML for the `Credential` object, as in:

    ```yaml
	apiVersion: k0rdent.mirantis.com/v1alpha1
	kind: Credential
	metadata:
	  name: accounting-cluster-credential
	  namespace: accounting
	spec:
  	  description: "Credentials for Accounting AWS account"
	  identityRef:
	    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
	    kind: AWSClusterStaticIdentity
	    name: accountingd-cluster-identity
	```

    As you can see, the `.spec.description` field gives more information about the `Credential`.

    If the `Credential` you need doesn't yet exist, you can ask your cloud administrator to create it, or you can
    follow the instructions in the [Credential System](admin-credentials.md), as well as the specific instructions for your [target infrastructure](admin-prepare.md), to crete it yourself.

    > Tip: 
    > Double-check to make sure that your credentials have sufficient permissions to create resources on the target infrastructure.

2. Select a Template

    Templates in k0rdent are predefined configurations that describe how to set up the cluster. Templates include details such as:
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
    kubectl describe clustertemplate aws-standalone-cp-0-0-5 -n kcm-system
    ```

3. Create a ClusterDeployment YAML Configuration

    Once you have the `Credential` and the `ClusterTemplate` you can create the `ClusterDeployment` object configuration. 
    It includes:
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

    This step submits your deployment request to k0rdent. 

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

7. Retrieve the Kubernetes Configuration

    When provisioning is complete, you can retrieve the kubeconfig file for the new cluster so you can interact with the cluster using `kubectl`:

    ```shell
    kubectl get secret -n <namespace> <cluster-name>-kubeconfig -o=jsonpath={.data.value} | base64 -d > kubeconfig
    ```
    You can then use this file to access the cluster, as in:

    ```shell
    export KUBECONFIG=kubeconfig
    kubectl get pods -A
    ```

    Store the kubeconfig file securely, as it contains authentication details for accessing the cluster.

## Cleanup

When you're finished you'll want to remove the cluster. Because the cluster is represented by the `ClusterDeployment` object,
deleting the cluster is a simple matter of deleting that object.  For example:

```shell
kubectl delete clusterdeployment <cluster-name> -n kcm-system
```

Note that even though the Kubernetes object is deleted immediately, it will take a few minutes for the actual resources to be removed.