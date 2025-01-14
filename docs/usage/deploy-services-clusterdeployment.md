# Deploy beach-head Services using Cluster Deployment

## Deployment

Beach-head services can be installed on a cluster deployment (i.e., target cluster) using the `ClusterDeployment` object.
Consider the following example:

> EXAMPLE: `ClusterDeployment` object for AWS Infrastructure Provider with beach-head services
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
>   services:
>     - template: kyverno-3-2-6
>       name: kyverno
>       namespace: kyverno
>     - template: ingress-nginx-4-11-3
>       name: ingress-nginx
>       namespace: ingress-nginx
>   servicesPriority: 100
>   stopOnConflict: false
> ```

In the example above the following fields are relevant to the deployment of beach-head services:

```yaml
  . . .
  services:
    - template: kyverno-3-2-6
      name: kyverno
      namespace: kyverno
    - template: ingress-nginx-4-11-3
      name: ingress-nginx
      namespace: ingress-nginx
  servicesPriority: 100
  stopOnConflict: false
  template: aws-standalone-cp-0-0-3
```

> NOTE:
> Refer to [Parameter List](#parameter-list) for more detail about these fields.

This example `ClusterDeployment` object will deploy kyverno and ingress-nginx referred to by their
service templates respectively on the target cluster it is managing. See the example below for
the service template for kyverno.

> EXAMPLE: `ServiceTemplate` object for kyverno version 3.2.6
> The `ServiceTemplate` for kyverno:
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ServiceTemplate
> metadata:
>   name: kyverno-3-2-6
>   namespace: kcm-system
> spec:
>   helm:
>     chartSpec:
>       chart: kyverno
>       interval: 10m0s
>       reconcileStrategy: ChartVersion
>       sourceRef:
>         kind: HelmRepository
>         name: kcm-templates
>       version: 3.2.6
> ```

The `kcm-templates` helm repository hosts the actual chart for kyverno version 3.2.6.
For more details see the [Bring your own Templates](../template/byo-templates.md) guide.

### Configuring Custom Values

Helm values can be passed to each beach-head services with the `.spec.services[].values` field in the `ClusterDeployment` or `MultiClusterService` object.

EXAMPLE: 
```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  . . .
  name: my-clusterdeployment
  namespace: kcm-system
  . . .
spec:
  . . .
  services:
   . . .
    - name: cert-manager
      namespace: cert-manager
      template: cert-manager-1-16-1
      values: |
        crds:
          enabled: true
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
        cert-manager:
          email: mail@example.net
    - template: ingress-nginx-4-11-3
      name: ingress-nginx
      namespace: ingress-nginx
   . . .
```

The example above shows how custom values are specified for each beach-head service.

### Templating Custom Values

Using Sveltos templating feature, we can also write templates which can be useful for automatically fetching pre-existing information within the cluster.

EXAMPLE:
```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-clusterdeployment
  namespace: kcm-system
spec:
  . . .
  servicesPriority: 100
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
```

In the example above the host and port information is being fetched from the spec of the CAPI cluster that belongs to this `ClusterDeployment`.

## Checking status

The `.status.services` field of the `ClusterDeployment` object shows the status for each of the beach-head services.

> EXAMPLE: Status for beach-head services deployed with `ClusterDeployment`
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   . . .
>   generation: 1
>   name: wali-aws-dev
>   namespace: kcm-system
>   . . .
> spec:
>   . . .
>   services:
>   - name: ingress-nginx
>     namespace: ingress-nginx
>     template: ingress-nginx-4-11-3
>   servicesPriority: 100
>   stopOnConflict: false
> status:
>   . . .
>   observedGeneration: 1
>   services:
>   - clusterName: my-managed-cluster
>     clusterNamespace: kcm-system
>     conditions:
>     - lastTransitionTime: "2024-12-11T23:03:05Z"
>       message: ""
>       reason: Provisioned
>       status: "True"
>       type: Helm
>     - lastTransitionTime: "2024-12-11T23:03:05Z"
>       message: Release kyverno/kyverno
>       reason: Managing
>       status: "True"
>       type: kyverno.kyverno/SveltosHelmReleaseReady
>     - lastTransitionTime: "2024-12-11T23:03:05Z"
>       message: Release ingress-nginx/ingress-nginx
>       reason: Managing
>       status: "True"
>       type: ingress-nginx.ingress-nginx/SveltosHelmReleaseReady
> ```

Based on the status above both kyverno and ingress-nginx should be installed in their respective namespaces on the target cluster.

```sh
➜  ~ kubectl get pod -n kyverno
NAME                                             READY   STATUS    RESTARTS   AGE
kyverno-admission-controller-96c5d48b4-sg5ts     1/1     Running   0          2m39s
kyverno-background-controller-65f9fd5859-tm2wm   1/1     Running   0          2m39s
kyverno-cleanup-controller-848b4c579d-ljrj5      1/1     Running   0          2m39s
kyverno-reports-controller-6f59fb8cd6-s8jc8      1/1     Running   0          2m39s
➜  ~ 
➜  ~ 
➜  ~ kubectl get pod -n ingress-nginx 
NAME                                       READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-cbcf8bf58-zhvph   1/1     Running   0          24m
```

> NOTE:
> * Refer to Step 7 of [Create Cluster Deployment](../usage/create-managed-cluster.md/step-7-retrieve-kubernetes-configuration-of-your-managed-cluster) guide for how to access the target cluster.
> * Refer to [Service Templates](../servicetemplates.md) for more detail on what statuses are reported.

## Removing beach-head services

To remove a beach-head service simply remove its entry from `.spec.services`.
The example below removes `kyverno-3-2-6` so its status also removed from `.status.services`.

> EAMPLE: Showing removal of `kyverno-3-2-6` from `ClusterDeployment`
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   . . .
>   generation: 2
>   name: wali-aws-dev
>   namespace: kcm-system
>   . . .
> spec:
>   . . .
>   services:
>   - name: ingress-nginx
>     namespace: ingress-nginx
>     template: ingress-nginx-4-11-3
>   servicesPriority: 100
>   stopOnConflict: false
> status:
>   . . .
>   observedGeneration: 2
>   services:
>   - clusterName: wali-aws-dev
>     clusterNamespace: kcm-system
>     conditions:
>     - lastTransitionTime: "2024-12-11T23:15:45Z"
>       message: ""
>       reason: Provisioned
>       status: "True"
>       type: Helm
>     - lastTransitionTime: "2024-12-11T23:15:45Z"
>       message: Release ingress-nginx/ingress-nginx
>       reason: Managing
>       status: "True"
>       type: ingress-nginx.ingress-nginx/SveltosHelmReleaseReady
> ```

## Parameter List

| Parameter                    | Example                | Description                                                                                   |
|------------------------------|------------------------|-----------------------------------------------------------------------------------------------|
| `.spec.servicesPriority`     | `100`                  | Sets the priority for the beach-head services defined in this spec (default: `100`)           |
| `.spec.stopOnConflict`       | `false`                | Stops deployment of beach-head services upon first encounter of a conflict (default: `false`) |
| `.spec.services[].template`  | `kyverno-3-2-6`        | Name of the `ServiceTemplate` object located in the same namespace                            |
| `.spec.services[].name`      | `my-kyverno-release`   | Release name for the beach-head service                                                       |
| `.spec.services[].namespace` | `my-kyverno-namespace` | Release namespace for the beach-head service (default: `.spec.services[].name`)               |
| `.spec.services[].values`    | `replicas: 3`          | Helm values to be used with the template while deployed the beach-head services               | 
| `.spec.services[].disable`   | `false`                | Disable handling of this beach-head service (default: `false`)                                |
