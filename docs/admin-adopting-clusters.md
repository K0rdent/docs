# Adopting an Existing Cluster

Creating a new cluster isn't the only way to use k0rdent. Adopting an existing Kubernetes cluster enables you to 
bring it under the k0rdent's management. This process is useful when you already have a running cluster but want 
to centralize management and leverage k0rdent's capabilities, such as unified monitoring, configuration, and automation.

To adopt a cluster, k0rdent establishes communication between the management cluster (where kcm is installed) 
and the target cluster. This requires proper credentials, network connectivity, and a standardized configuration. 

Follow these steps to adopt an existing cluster:

1. Prerequisites

    Before you start, make sure you have the following:

    - A kubeconfig file for the cluster you want to adopt (this file provides access credentials and configuration details 
      for the cluster).
    - A management cluster with k0rdent installed and running. See the [installation instructions](admin-installation.md) 
      if you need to set it up.
    - Network connectivity between the management cluster and the cluster to be adopted (for example, ensure firewall 
      rules and VPNs allow communication).

2.  Create a Credential

    Start by creating a `Credential` object that includes all required authentication details for your chosen infrastructure 
    provider. Follow the instructions in the [Credential System](admin-credentials.md), as well as the specific instructions 
    for your [target infrastructure](admin-prepare.md).

    > TIP:  
    > Double-check that your credentials have sufficient permissions to create resources on the target infrastructure.

3. Configure the Adopted Cluster Template

    Set the `KUBECONFIG` environment variable to the path of your management cluster's kubeconfig file so you can 
    execute commands against the management cluster.

    For example:

    ```shell
    export KUBECONFIG=/path/to/management-cluster-kubeconfig
    ```

4. Create the `ClusterDeployment` YAML Configuration

    The `ClusterDeployment` object is used to define how k0rdent should manage the adopted cluster. Create a 
    YAML file for the `ClusterDeployment` object, as shown below:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: <CLUSTER_NAME>
      namespace: <NAMESPACE>
    spec:
      template: adopted-cluster-<VERSION>
      credential: <CREDENTIAL_NAME>
      dryRun: <BOOLEAN>
      config:
        <CONFIGURATION>
    ```

    Replace placeholders like `<CLUSTER_NAME>`, `<NAMESPACE>`, `<VERSION>`, `<CREDENTIAL_NAME>`, and `<CONFIGURATION>` with actual values. The `dryRun` flag is useful for testing the configuration without making changes to the cluster. For more details, see the [Dry Run](appendix-dryrun.md) section.

    You can also get a list of the available templates with:

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

    Putting it all together, your YAML would look something like this:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: my-cluster
      namespace: kcm-system
    spec:
      template: adopted-cluster-0-0-2
      credential: my-cluster-credential
      dryRun: false
      config: {}
    ```

5. Apply the `ClusterDeployment` configuration

    Once your configuration file is ready, apply it to the management cluster using `kubectl`:

    ```shell
    kubectl apply -f clusterdeployment.yaml
    ```

    This step submits the `ClusterDeployment` object to k0rdent, initiating the adoption process.

6. Check the Status of the `ClusterDeployment` Object

    To ensure the adoption process is progressing as expected, check the status of the `ClusterDeployment` object:

    ```shell
    kubectl -n <namespace> get clusterdeployment.kcm <cluster-name> -o=yaml
    ```

    The output includes the current state and any conditions (for example, errors or progress updates). Review 
    this information to confirm that the adoption is successful.

## What's Happening Behind the Scenes?

When you adopt a cluster, k0rdent performs several actions:
1. It validates the credentials and configuration provided in the `ClusterDeployment` object.
2. It ensures network connectivity between the management cluster and the adopted cluster.
3. It registers the adopted cluster within the k0rdent system, enabling it to be monitored and managed like 
   any k0rdent-deployed cluster.

This process doesn't change the adopted cluster's existing workloads or configurations. Instead, it enhances your 
ability to manage the cluster through k0rdent.

## Additional Tips
- If you encounter issues, double-check that kubeconfig file you used for the adopted cluster is valid 
  and matches the cluster you're trying to adopt.
- Use the [`dryRun`](appendix-dryrun.md) option during the first attempt to validate the configuration without making actual changes.