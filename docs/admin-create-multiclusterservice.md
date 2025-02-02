# Deploy beach-head services using MultiClusterService
The `MultiClusterService` object is used to deploy beach-head services on multiple matching clusters.
## Creation
The `MultiClusterService` object can be created with the following YAML:
```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: MultiClusterService
metadata:
  name: <name>
spec:
  clusterSelector:
    matchLabels:
      <key1>: <value1>
      <key2>: <value2>
      . . .
 serviceSpec:
    services:
    - template: <servicetemplate-1-name>
      name: <release-name>
      namespace: <release-namespace>
    priority: 100
```
## Matching Multiple Clusters

Consider the following example where 2 clusters have been deployed using ClusterDeployment objects:

Command:

```shell
kubectl get clusterdeployments.k0rdent.mirantis.com -n kcm-system
```

Output:
```shell
NAME             READY   STATUS
dev-cluster-1   True    ClusterDeployment is ready
dev-cluster-2   True    ClusterDeployment is ready
```

Command:
```shell
 kubectl get cluster -n kcm-system --show-labels
```
Output:
```shell
NAME           CLUSTERCLASS     PHASE         AGE     VERSION   LABELS
dev-cluster-1                  Provisioned   2h41m             app.kubernetes.io/managed-by=Helm,helm.toolkit.fluxcd.io/name=dev-cluster-1,helm.toolkit.fluxcd.io/namespace=kcm-system,sveltos-agent=present
dev-cluster-2                  Provisioned   3h10m             app.kubernetes.io/managed-by=Helm,helm.toolkit.fluxcd.io/name=dev-cluster-2,helm.toolkit.fluxcd.io/namespace=kcm-system,sveltos-agent=present
```
> EXAMPLE: 
> Spec for `dev-cluster-1` ClusterDeployment (only sections relevant to beach-head services):
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   name: dev-cluster-1
>   namespace: kcm-system
> spec:
>   . . .
>   serviceSpec:
>     services:
>     - name: kyverno
>       namespace: kyverno
>       template: kyverno-3-2-6
>     - name: ingress-nginx
>       namespace: ingress-nginx
>       template: ingress-nginx-4-11-0
>     priority: 100
>   . . .
> ```
>
> Spec for `dev-cluster-2` ClusterDeployment (only sections relevant to beach-head services):
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   name: dev-cluster-2
>   namespace: kcm-system
> spec:
>   . . .
>   serviceSpec:
>     services:
>     - name: ingress-nginx
>       namespace: ingress-nginx
>       template: ingress-nginx-4-11-0
>     priority: 500
>   . . .
> ```

> NOTE:
> See [Deploy beach-head Services using Cluster Deployment](user-create-service.md) for how to use beach-head services with ClusterDeployment.
> Now the following `global-ingress` MultiClusterService object is created with the following spec:
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: MultiClusterService
> metadata:
>   name: global-ingress
> spec:
>   clusterSelector:
>     matchLabels:
>       app.kubernetes.io/managed-by: Helm
>   serviceSpec:
>     services:
>     - name: ingress-nginx
>       namespace: ingress-nginx
>       template: ingress-nginx-4-11-3
>     priority: 300
> ```
>
> This MultiClusterService will match any CAPI cluster with the label `app.kubernetes.io/managed-by: Helm` and deploy chart
> version 4.11.3 of ingress-nginx service on it.

### Configuring Custom Values

Refer to "Configuring Custom Values" in [Deploy beach-head Services using Cluster Deployment](user-create-service.md#deployment-of-beach-head-services).
### Templating Custom Values
Refer to "Templating Custom Values" in [Deploy beach-head Services using Cluster Deployment](user-create-service.md#configuring-custom-values).
### Services Priority and Conflict

The `.spec.serviceSpec.priority` field is used to specify the priority for the services managed by a ClusterDeployment or MultiClusterService object.
Considering the example above:

1. ClusterDeployment `dev-cluster-1` manages deployment of kyverno (v3.2.6) and ingress-nginx (v4.11.0) with `priority=100` on its cluster.
2. ClusterDeployment `dev-cluster-2` manages deployment of ingress-nginx (v4.11.0) with `priority=500` on its cluster.
3. MultiClusterService `global-ingress` manages deployment of ingress-nginx (v4.11.3) with `priority=300` on both clusters.
This scenario presents a conflict on both the clusters as the MultiClusterService is attempting to deploy v4.11.3 of ingress-nginx
on both whereas the ClusterDeployment for each is attempting to deploy v4.11.0 of ingress-nginx.

This is where `.spec.serviceSpec.priority` can be used to specify who gets the priority. Higher number means higer priority and vice versa. In this example:
1. MultiClusterService "global-ingress" will take precedence over ClusterDeployment "dev-cluster-1" and ingress-nginx (v4.11.3) defined in MultiClusterService object will be deployed on the cluster.
2. ClusterDeployment "dev-cluster-2" will take precedence over MultiClusterService "global-ingress" and ingress-nginx (v4.11.0) defined in ClusterDeployment object will be deployed on the cluster.

> NOTE: If `priority` are equal, the first one to reach the cluster wins and deploys its beach-head services.

## Checking Status

The status for the MultiClusterService object will show the deployment status for the beach-head services managed
by it on each of the CAPI target clusters that it matches. Consider the same example where 2 ClusterDeployments
and 1 MultiClusterService is deployed.

> EXAMPLE: Status for `global-ingress` MultiClusterService
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: MultiClusterService
> metadata:
>   . . .
>   name: global-ingress
>   resourceVersion: "38146"
>   . . .
> spec:
>   clusterSelector:
>     matchLabels:
>       app.kubernetes.io/managed-by: Helm
>   serviceSpec:
>     services:
>     - name: ingress-nginx
>       namespace: ingress-nginx
>       template: ingress-nginx-4-11-3
>     . . .
>   . . .
> status:
>   conditions:
>   - lastTransitionTime: "2024-10-25T08:36:24Z"
>     message: ""
>     reason: Succeeded
>     status: "True"
>     type: SveltosClusterProfileReady
>   - lastTransitionTime: "2024-10-25T08:36:24Z"
>     message: MultiClusterService is ready
>     reason: Succeeded
>     status: "True"
>     type: Ready
>   observedGeneration: 1
>   services:
>   - clusterName: dev-cluster-2
>     clusterNamespace: kcm-system
>     conditions:
>     - lastTransitionTime: "2024-10-25T08:36:35Z"
>       message: |
>         cannot manage chart ingress-nginx/ingress-nginx. ClusterSummary p--dev-cluster-2-capi-dev-cluster-2 managing it.
>       reason: Failed
>       status: "False"
>       type: Helm
>     - lastTransitionTime: "2024-10-25T08:36:25Z"
>       message: 'Release ingress-nginx/ingress-nginx: ClusterSummary p--dev-cluster-2-capi-dev-cluster-2
>         managing it'
>       reason: Conflict
>       status: "False"
>       type: ingress-nginx.ingress-nginx/SveltosHelmReleaseReady
>   - clusterName: dev-cluster-1
>     clusterNamespace: kcm-system
>     conditions:
>     - lastTransitionTime: "2024-10-25T08:36:24Z"
>       message: ""
>       reason: Provisioned
>       status: "True"
>       type: Helm
>     - lastTransitionTime: "2024-10-25T08:36:25Z"
>       message: Release ingress-nginx/ingress-nginx
>       reason: Managing
>       status: "True"
>       type: ingress-nginx.ingress-nginx/SveltosHelmReleaseReady
> ```

The status under `.status.services` shows a conflict for `dev-cluster-2` as expected because the MultiClusterService has a lower priority.
Whereas, it shows provisioned for `dev-cluster-1` because the MultiClusterService has a higher priority.

> EXAMPLE: Status for `dev-cluster-1` ClusterDeployment (only sections relevant to beach-head services):
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   . . . 
>   name: dev-cluster-1
>   namespace: kcm-system
>   . . .
> spec:
>   . . .
>   serviceSpec:
>     services:
>     - name: kyverno
>       namespace: kyverno
>       template: kyverno-3-2-6
>     - name: ingress-nginx
>       namespace: ingress-nginx
>       template: ingress-nginx-4-11-0
>     priority: 100
>     . . .
>   . . .
> status:
>   . . .
>   services:
>   - clusterName: dev-cluster-1
>     clusterNamespace: kcm-system
>     conditions:
>     - lastTransitionTime: "2024-10-25T08:36:35Z"
>       message: |
>         cannot manage chart ingress-nginx/ingress-nginx. ClusterSummary global-ingress-capi-dev-cluster-1 managing it.
>       reason: Provisioning
>       status: "False"
>       type: Helm
>     - lastTransitionTime: "2024-10-25T07:44:43Z"
>       message: Release kyverno/kyverno
>       reason: Managing
>       status: "True"
>       type: kyverno.kyverno/SveltosHelmReleaseReady
>     - lastTransitionTime: "2024-10-25T08:36:25Z"
>       message: 'Release ingress-nginx/ingress-nginx: ClusterSummary global-ingress-capi-dev-cluster-1
>         managing it'
>       reason: Conflict
>       status: "False"
>       type: ingress-nginx.ingress-nginx/SveltosHelmReleaseReady
> ```

The status under `.status.services` for ClusterDeployment `dev-cluster-1` shows that it is managing Kyverno but unable to manage ingress-nginx because
another object with higher priority is managing it, so it shows a conflict instead.

> EXAMPLE: Status for `dev-cluster-2` ClusterDeployment (only sections relevant to beach-head services):
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterDeployment
> metadata:
>   . . .
>   name: dev-cluster-2
>   namespace: kcm-system
>   resourceVersion: "30889"
>   . . .
> spec:
>   . . .
>   serviceSpec:
>     services:
>     - name: ingress-nginx
>       namespace: ingress-nginx
>       template: ingress-nginx-4-11-0
>     priority: 500
>     . . .
>   . . .
> status:
>   . . .
>   services:
>   - clusterName: dev-cluster-2
>     clusterNamespace: kcm-system
>     conditions:
>     - lastTransitionTime: "2024-10-25T08:18:22Z"
>       message: ""
>       reason: Provisioned
>       status: "True"
>       type: Helm
>     - lastTransitionTime: "2024-10-25T08:18:22Z"
>       message: Release ingress-nginx/ingress-nginx
>       reason: Managing
>       status: "True"
>       type: ingress-nginx.ingress-nginx/SveltosHelmReleaseReady
> ```

The status under `.status.services` for ClusterDeployment `dev-cluster-2` shows that it is managing ingress-nginx as expected since it has a higher priority.

## Parameter List
Refer to "Parameter List" in [Deploy beach-head Services using Cluster Deployment](user-create-service.md#deployment-of-beach-head-services).