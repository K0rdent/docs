# Create Cluster Deployment

## Creation Process 

### Step 1: Create Credential

- Create a `Credential` object with all credentials required per the
  [Credential System](../credential/main.md).

### Step 2: Select the Template

For details about the templates in k0rdent, see the [Templates system](../template/main.md).

- Set the `KUBECONFIG` environment variable to the path to the management
  cluster kubeconfig file. Then select the `Template` you want to use for the
  deployment. To list all available templates, run:

  ```shell
  kubectl get clustertemplate -n kcm-system
  ```

> NOTE:
>
> If you want to deploy a hosted control plane template, check additional notes
> on hosted control planes for each of the clustertemplate sections:
>
> - [AWS Hosted Control Plane](../clustertemplates/aws/hosted-control-plane.md)
> - [vSphere Hosted Control Plane](../clustertemplates/vsphere/hosted-control-plane.md)

### Step 3: Create the ClusterDeployment Object YAML Configuration

- Create the file with the `ClusterDeployment` configuration:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterDeployment
    metadata:
      name: <cluster-name>
      namespace: <kcm-system-namespace>
    spec:
      template: <template-name>
      credential: <infrastructure-provider-credential-name>
      dryRun: <"true" or "false": defaults to "false">
      config:
        <cluster-configuration>
    ```

> NOTE:
>
> Substitute the parameters enclosed in angle brackets with the corresponding
> values. Enable the `dryRun` flag if required. For details, see
> [Dry Run](#dry-run).

Following is an interpolated example.

> EXAMPLE: `ClusterDeployment` for AWS Infrastructure Provider Object Example
> 
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   name: my-managed-cluster
>   namespace: kcm-system
> spec:
>   template: aws-standalone-cp-0-0-3
>   credential: aws-credential
>   dryRun: true
>   config:
>     region: us-west-2
>     controlPlane:
>       instanceType: t3.small
>     worker:
>       instanceType: t3.small
> ```

### Step 4: Apply the `ClusterDeployment` Configuration to Create it

- Apply the `ClusterDeployment` object to your k0rdent deployment:

	```shell
	kubectl apply -f clusterdeployment.yaml
	```

### Step 5: Check the Status of the `ClusterDeployment` Object

- Check the status of the newly created `ClusterDeployment`:

	```shell
	kubectl -n <namespace> get clusterdeployment.kcm <cluster-name> -o=yaml
	```

> INFO:
> 
> Reminder: `<namespace>` and `<cluster-name>` are defined in the `.metadata`
> section of the `ClusterDeployment` object you created above.

### Step 6: Wait for Infrastructure and Cluster to be Provisioned

- Wait for infrastructure to be provisioned and the cluster to be deployed:

	```shell
	kubectl -n <namespace> get cluster <cluster-name> -o=yaml
	```

> TIP:
> 
> You may also watch the process with the `clusterctl describe` command
> (requires the `clusterctl` CLI to be installed):
> 
> ```shell
> clusterctl describe cluster <cluster-name> -n <namespace> --show-conditions all
> ```

### Step 7: Retrieve Kubernetes Configuration of Your Cluster Deployment

- Retrieve the Kubernetes configuration of your cluster deployment when it is
  finished provisioning:

    ```shell
    kubectl get secret -n <namespace> <cluster-name>-kubeconfig -o=jsonpath={.data.value} | base64 -d > kubeconfig
    ```

## Dry Run

k0rdent `ClusterDeployment` supports two modes: with and without `.spec.dryRun`
(defaults to `false`).

If no configuration (`.spec.config`) is specified, the `ClusterDeployment` object
will be populated with defaults (default configuration can be found in the
corresponding `Template` status) and automatically have `.spec.dryRun` set to
`true`.

> EXAMPLE: `ClusterDeployment` with default configuration
> 
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   name: my-managed-cluster
>   namespace: kcm-system
> spec:
>   config:
>     clusterNetwork:
>       pods:
>         cidrBlocks:
>         - 10.244.0.0/16
>       services:
>         cidrBlocks:
>         - 10.96.0.0/12
>     controlPlane:
>       iamInstanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
>       instanceType: ""
>     controlPlaneNumber: 3
>     k0s:
>       version: v1.27.2+k0s.0
>     publicIP: false
>     region: ""
>     sshKeyName: ""
>     worker:
>       amiID: ""
>       iamInstanceProfile: nodes.cluster-api-provider-aws.sigs.k8s.io
>       instanceType: ""
>     workersNumber: 2
>   template: aws-standalone-cp-0-0-3
>   credential: aws-credential
>   dryRun: true
> ```

After you adjust your configuration and ensure that it passes validation
(`TemplateReady` condition from `.status.conditions`), remove the `.spec.dryRun`
flag to proceed with the deployment.

Here is an example of a `ClusterDeployment` object that passed the validation:

> EXAMPLE: `ClusterDeployment` object that passed the validation
> 
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   name: my-managed-cluster
>   namespace: kcm-system
> spec:
>   template: aws-standalone-cp-0-0-3
>   credential: aws-credential
>   config:
>     region: us-east-2
>     publicIP: true
>     controlPlaneNumber: 1
>     workersNumber: 1
>     controlPlane:
>       instanceType: t3.small
>     worker:
>       instanceType: t3.small
>   status:
>     conditions:
>     - lastTransitionTime: "2024-07-22T09:25:49Z"
>       message: Template is valid
>       reason: Succeeded
>       status: "True"
>       type: TemplateReady
>     - lastTransitionTime: "2024-07-22T09:25:49Z"
>       message: Helm chart is valid
>       reason: Succeeded
>       status: "True"
>       type: HelmChartReady
>     - lastTransitionTime: "2024-07-22T09:25:49Z"
>       message: ClusterDeployment is ready
>       reason: Succeeded
>       status: "True"
>       type: Ready
>     observedGeneration: 1
> ```

<!-- This Cleanup section describes uninstalling k0rdent from the super cluster and hence should be in its own file. -->

## Cleanup

1. Remove the Management object:

	```shell
	kubectl delete management.kcm kcm
	```

> NOTE:
>
> Ensure you have no k0rdent `ClusterDeployment` objects left in the cluster
> prior to Management deletion.

2. Remove the `kcm` Helm release:

	```shell
	helm uninstall kcm -n kcm-system
	```

3. Remove the `kcm-system` namespace:

	```shell
	kubectl delete ns kcm-system
	```
