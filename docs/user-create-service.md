# Deploy Services to a Managed Cluster

AT its heart, everything in k0rdent is based on templates that help define Kubernetes objects. For clusters,
these are `ClusterTemplate`s. For applications and services, these are `ServiceTemplate`s.

## Understanding `ServiceTemplate`s

`ServiceTemplate`s are meant to let k0rdent know where to find a Helm chart with instructions for installing
an application. In many cases, these charts will be in a private repository.  For example, consider this template for
installing the Nginx Ingress:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ServiceTemplate
metadata:
  name: project-ingress-nginx-4.11.0
  namespace: tenant42
spec:
  helm:
    chartSpec:
      chart: demo-ingress-nginx
      version: 4.11.0
      interval: 10m0s
      sourceRef:
        kind: HelmRepository
        name: k0rdent-demos
---
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ServiceTemplateChain
metadata:
  name: project-ingress-nginx-4.11.0
  namespace: tenant42
spec:
  supportedTemplates:
    - name: project-ingress-nginx-4.11.0
    - name: project-ingress-nginx-4.10.0
      availableUpgrades:
        - name: project-ingress-nginx-4.11.0
```

Here you see a template called `project-ingress-nginx-4.11.0` that is meant to be deployed in the `tenant42` namespace.
The `.spec.helm.chartSpec` specifies the name of the Helm chart and where to find it, as well as the version and other 
important information. The `ServiceTemplateChain` shows that this template is also an upgrade path from version 4.10.0.

If you wanted to deploy this as an application, you would first go ahead and add it to the cluster in which you were
working, so if you were to save this YAML to a file called `project-ingress.yaml` you could run this command on the management cluster:

```shell
kubectl apply -f project-ingress.yaml -n tenant42
```
## Adding a `Service` to a `ClusterDeployment`

To add the service defined by this template to a cluster, you would simply add it to the `ClusterDeployment` object
when you create it, as in:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-managed-cluster
  namespace: tenant42
spec:
  config:
    clusterLabels: {}
  template: aws-standalone-cp-0-0-3
  credential: aws-credential
  services:
    - template: project-ingress-nginx-4.11.0
      name: ingress-nginx
      namespace: tenant42
  servicesPriority: 100
  stopOnConflict: false
```
As you can see, you're simply referencing the template in the `.spec.services.template` field of the `ClusterDeployment`
to tell k0rdent that you want this service to be part of this cluster.

If you wanted to add this serice to an existing cluster, you would simply patch the definition of the `ClusterDeployment`, as in:

```yaml
kubectl patch clusterdeployment my-managed-cluster -n tenant42 --type='merge' -p '
spec:
  services:
    - template: project-ingress-nginx-4.11.0
      name: ingress-nginx
      namespace: tenant42
```

Let's look at a more complex case, involving deploying beach-head services on a single cluster.

## Deployment of beach-head services

Beach-head services can be installed on a cluster deployment (that is, a target cluster) using the `ClusterDeployment` object, just as with a single service. Consider the following example of a `ClusterDeployment` object for AWS Infrastructure Provider with beach-head services

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-managed-cluster
  namespace: kcm-system
spec:
  config:
    clusterLabels: {}
    clusterIdentity:
      name: aws-cluster-identity
      namespace: kcm-system
    controlPlane:
      amiID: ami-0eb9fdcf0d07bd5ef
      instanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
      instanceType: t3.small
      controlPlaneNumber: 1
      publicIP: true
      region: ca-central-1
    worker:
      amiID: ami-0eb9fdcf0d07bd5ef
      instanceProfile: nodes.cluster-api-provider-aws.sigs.k8s.io
      instanceType: t3.small
    workersNumber: 1
  credential: aws-cluster-identity-cred
  serviceSpec:
    services:
      - template: kyverno-3-2-6
        name: kyverno
        namespace: kyverno
      - template: ingress-nginx-4-11-3
        name: ingress-nginx
        namespace: ingress-nginx
    priority: 100
  template: aws-standalone-cp-0-0-6
```

In the example above the fields under `serviceSpec` are relevant to the deployment of beach-head services.

> NOTE:
> Refer to the [Template Guide](template-intro.md) for more detail about these fields.

This example `ClusterDeployment` object deploys kyverno and ingress-nginx, as referred to by their
service templates respectively, on the target cluster.  As before, the `ServiceTemplate` includes information on the service. For example, here is the `ServiceTemplate` for kyverno:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ServiceTemplate
metadata:
  name: kyverno-3-2-6
  annotations:
    helm.sh/resource-policy: keep
spec:
  helm:
    chartSpec:
      chart: kyverno
      version: 3.2.6
      interval: 10m0s
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: kcm-templates
      version: 3.2.6
        name: k0rdent-templates
```

The `k0rdent-templates` helm repository hosts the actual kyverno chart version 3.2.6.
For more details see the [Bring your own Templates](template-byo.md) guide.

### Configuring Custom Values

Helm values can be passed to each beach-head services with the `.spec.serviceSpec.services[].values` field in the `ClusterDeployment` or `MultiClusterService` object. For example:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-clusterdeployment
  namespace: kcm-system
spec:
  . . .
  serviceSpec:
    services:
    - template: ingress-nginx-4-11-3
      name: ingress-nginx
      namespace: ingress-nginx
      values: |
        ingress-nginx:
          controller:
            replicaCount: 3
    - name: kyverno
      namespace: kyverno
      template: kyverno-3-2-6
      values: |
        kyverno:
          admissionController:
            replicas: 3
    - name: motel-regional
      namespace: motel
      template: motel-regional-0-1-1
      values: |
        victoriametrics:
          vmauth:
            ingress:
              host: vmauth.kcm0.example.net
            credentials:
              username: motel
              password: motel
        grafana:
          ingress:
            host: grafana.kcm0.example.net
   . . .
```
> NOTE: The values for ingress-nginx and kyverno start with the "ingress-nginx:" and "kyverno:" keys respectively because
> the helm charts used by the ingress-nginx-4-11-3 and kyverno-3-2-6 `ServiceTemplates` use the official upstream
> helm charts for ingress-nginx and kyverno as dependencies.

### Templating Custom Values

Using the Sveltos templating feature, we can also write templates that can be useful for automatically fetching pre-existing information within the cluster. For example:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-clusterdeployment
  namespace: kcm-system
spec:
  . . .
  serviceSpec:
    services:
    - template: motel-0-1-0
      name: motel
      namespace: motel
    - template: myappz-0-3-0
      name: myappz
      namespace: myappz
      values: |
        controlPlaneEndpointHost: {{ .Cluster.spec.controlPlaneEndpoint.host }}
        controlPlaneEndpointPort: "{{ .Cluster.spec.controlPlaneEndpoint.port }}"
    priority: 100
    . . .        
```

In this case, the host and port information will be fetched from the spec of the CAPI cluster that hosts this `ClusterDeployment`.

## Checking status

The `.status.services` field of the `ClusterDeployment` object shows the status for each of the beach-head services.
For example, if you were to `describe` the `ClusterDeployment` with these services, you would see `condition`s that show
status information, as in:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  . . .
  generation: 1
  name: wali-aws-dev
  namespace: kcm-system
  . . .
spec:
  . . .
  serviceSpec:
    services:
    - name: ingress-nginx
      namespace: ingress-nginx
      template: ingress-nginx-4-11-3
    - name: kyverno
      namespace: kyverno
      template: kyverno-3-2-6
    . . .
status:
  . . .
  observedGeneration: 1
  services:
  - clusterName: my-managed-cluster
    clusterNamespace: kcm-system
    conditions:
    - lastTransitionTime: "2024-12-11T23:03:05Z"
      message: ""
      reason: Provisioned
      status: "True"
      type: Helm
    - lastTransitionTime: "2024-12-11T23:03:05Z"
      message: Release kyverno/kyverno
      reason: Managing
      status: "True"
      type: kyverno.kyverno/SveltosHelmReleaseReady
    - lastTransitionTime: "2024-12-11T23:03:05Z"
      message: Release ingress-nginx/ingress-nginx
      reason: Managing
      status: "True"
      type: ingress-nginx.ingress-nginx/SveltosHelmReleaseReady
```

Based on the information above both kyverno and ingress-nginx are installed in their respective namespaces on the target cluster.
You can check to see for yourself:

```shell
kubectl get pod -n kyverno
```
```console
NAME                                             READY   STATUS    RESTARTS   AGE
kyverno-admission-controller-96c5d48b4-sg5ts     1/1     Running   0          2m39s
kyverno-background-controller-65f9fd5859-tm2wm   1/1     Running   0          2m39s
kyverno-cleanup-controller-848b4c579d-ljrj5      1/1     Running   0          2m39s
kyverno-reports-controller-6f59fb8cd6-s8jc8      1/1     Running   0          2m39s
```
```shell
kubectl get pod -n ingress-nginx 
```
```console
NAME                                       READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-cbcf8bf58-zhvph   1/1     Running   0          24m
```

Youc an get more information on how to access the managed cluster in the [create a cluster deployment](admin-creating-clusters.md)
chapter, and more on `ServiceTemplate`s in the [Template Guide](template-intro.md).

## Removing beach-head services

To remove a beach-head service simply remove its entry from `.spec.serviceSpec.services`.
The example below removes `kyverno-3-2-6`, so its status also removed from `.status.services`.

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  . . .
  generation: 2
  name: wali-aws-dev
  namespace: kcm-system
  . . .
spec:
  . . .
  serviceSpec:
    services:
    - name: ingress-nginx
      namespace: ingress-nginx
      template: ingress-nginx-4-11-3
    priority: 100
    . . .
status:
  . . .
  observedGeneration: 2
  services:
  - clusterName: wali-aws-dev
    clusterNamespace: kcm-system
    conditions:
    - lastTransitionTime: "2024-12-11T23:15:45Z"
      message: ""
      reason: Provisioned
      status: "True"
      type: Helm
    - lastTransitionTime: "2024-12-11T23:15:45Z"
      message: Release ingress-nginx/ingress-nginx
      reason: Managing
      status: "True"
      type: ingress-nginx.ingress-nginx/SveltosHelmReleaseReady
```

## Parameter List

Here is an idea of the parameters involved.

| Parameter                                 | Example                | Description                                                                                   |
|-------------------------------------------|------------------------|-----------------------------------------------------------------------------------------------|
| `.spec.serviceSpec.syncMode`              | `Continuous`           | Specifies how beach-head services are synced i the target cluster (default:`Continuous`)      |
| `.spec.serviceSpec.DriftIgnore`           |                        | specifies resources to ignore for drift detection                                             |
| `.spec.serviceSpec.DriftExclusions`       |                        | specifies specific configurations of resources to ignore for drift detection                  |
| `.spec.serviceSpec.priority`              | `100`                  | Sets the priority for the beach-head services defined in this spec (default: `100`)           |
| `.spec.serviceSpec.stopOnConflict`        | `false`                | Stops deployment of beach-head services upon first encounter of a conflict (default: `false`) |
| `.spec.serviceSpec.services[].template`   | `kyverno-3-2-6`        | Name of the `ServiceTemplate` object located in the same namespace                            |
| `.spec.serviceSpec.services[].name`       | `my-kyverno-release`   | Release name for the beach-head service                                                       |
| `.spec.serviceSpec.services[].namespace`  | `my-kyverno-namespace` | Release namespace for the beach-head service (default: `.spec.services[].name`)               |
| `.spec.serviceSpec.services[].values`     | `replicas: 3`          | Helm values to be used with the template while deployed the beach-head services               |
| `.spec.serviceSpec.services[].valuesFrom` | ``                     | Can reference a ConfigMap or Secret containing helm values                                    |
| `.spec.serviceSpec.services[].disable`    | `false`                | Disable handling of this beach-head service (default: `false`)                                |