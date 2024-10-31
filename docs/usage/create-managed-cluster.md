To deploy a managed cluster:

1. Select the `Template` you want to use for the deployment. To list all available templates, run:

```bash
export KUBECONFIG=<path-to-management-kubeconfig>

kubectl get template -n hmc-system -o go-template='{{ range .items }}{{ if eq .status.type "deployment" }}{{ .metadata.name }}{{ printf "\n" }}{{ end }}{{ end }}'
```

For details about the `Template system` in HMC, see [Templates system](../template/main.md).

If you want to deploy hostded control plate template, make sure to check additional notes on Hosted control plane for each of the clustertemplate sections.

2. Create the file with the `ManagedCluster` configuration:

> NOTE:
> Substitute the parameters enclosed in angle brackets with the corresponding values.
> Enable the `dryRun` flag if required. For details, see [Dry run](#dry-run).

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: ManagedCluster
metadata:
  name: <cluster-name>
  namespace: <cluster-namespace>
spec:
  template: <template-name>
  dryRun: <true/false>
  config:
    <cluster-configuration>
```

3. Create the `ManagedCluster` object:

`kubectl create -f managedcluster.yaml`

4. Check the status of the newly created `ManagedCluster` object:

`kubectl -n <managedcluster-namespace> get managedcluster.hmc <managedcluster-name> -o=yaml`

5. Wait for infrastructure to be provisioned and the cluster to be deployed (the provisioning starts only when
`spec.dryRun` is disabled):

   `kubectl -n <managedcluster-namespace> get cluster <managedcluster-name> -o=yaml`

> TIP:
> You may also watch the process with the `clusterctl describe` command (requires the `clusterctl` CLI to be installed):
> ```
> clusterctl describe cluster <managedcluster-name> -n <managedcluster-namespace> --show-conditions all
> ```

6. Retrieve the `kubeconfig` of your managed cluster:

```
kubectl get secret -n hmc-system <managedcluster-name>-kubeconfig -o=jsonpath={.data.value} | base64 -d > kubeconfig
```

### Dry run

HMC `ManagedCluster` supports two modes: with and without (default) `dryRun`.

If no configuration (`spec.config`) provided, the `ManagedCluster` object will be populated with defaults
(default configuration can be found in the corresponding `Template` status) and automatically marked as `dryRun`.

Here is an example of the `ManagedCluster` object with default configuration:

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: ManagedCluster
metadata:
  name: <cluster-name>
  namespace: <cluster-namespace>
spec:
  config:
    clusterNetwork:
      pods:
        cidrBlocks:
        - 10.244.0.0/16
      services:
        cidrBlocks:
        - 10.96.0.0/12
    controlPlane:
      amiID: ""
      iamInstanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
      instanceType: ""
    controlPlaneNumber: 3
    k0s:
      version: v1.27.2+k0s.0
    publicIP: false
    region: ""
    sshKeyName: ""
    worker:
      amiID: ""
      iamInstanceProfile: nodes.cluster-api-provider-aws.sigs.k8s.io
      instanceType: ""
    workersNumber: 2
  template: aws-standalone-cp
  dryRun: true
```

After you adjust your configuration and ensure that it passes validation (`TemplateReady` condition
from `status.conditions`), remove the `spec.dryRun` flag to proceed with the deployment.

Here is an example of a `ManagedCluster` object that passed the validation:

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: ManagedCluster
metadata:
  name: aws-standalone
  namespace: aws
spec:
  template: aws-standalone-cp
  config:
    region: us-east-2
    publicIP: true
    controlPlaneNumber: 1
    workersNumber: 1
    controlPlane:
      amiID: ami-02f3416038bdb17fb
      instanceType: t3.small
    worker:
      amiID: ami-02f3416038bdb17fb
      instanceType: t3.small
  status:
    conditions:
    - lastTransitionTime: "2024-07-22T09:25:49Z"
      message: Template is valid
      reason: Succeeded
      status: "True"
      type: TemplateReady
    - lastTransitionTime: "2024-07-22T09:25:49Z"
      message: Helm chart is valid
      reason: Succeeded
      status: "True"
      type: HelmChartReady
    - lastTransitionTime: "2024-07-22T09:25:49Z"
      message: ManagedCluster is ready
      reason: Succeeded
      status: "True"
      type: Ready
    observedGeneration: 1
```

## Cleanup

1. Remove the Management object:

`kubectl delete management.hmc hmc`

> NOTE:
> Make sure you have no HMC ManagedCluster objects left in the cluster prior to Management deletion

2. Remove the `hmc` Helm release:

`helm uninstall hmc -n hmc-system`

3. Remove the `hmc-system` namespace:

`kubectl delete ns hmc-system`
