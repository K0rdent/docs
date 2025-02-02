# Detecting and Correcting Drift

## Enabling Drift Detection

Set `.spec.serviceSpec.syncMode=Continuous` in the `ClusterDeployment` or `MultiClusterService` object to enable drift detection and correction. Sveltos will then automatically deploy the drift-detection-manager on the targeted clusters:

```sh
kubectl -n projectsveltos get deployments.apps 
```
```console
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
drift-detection-manager   1/1     1            1           152m
sveltos-agent-manager     1/1     1            1           152m
```

## How Drift Detection and Correction Works

The drift-detection-manager watches for the deployed helm chart resources (i.e., the resources deployed via a `ServiceTemplate`) and if it detects any changes 
in the spec of the resources based on hash value, it updates the status of the `ResourceSummary` object. This change triggers the addon-controller in the "projectsveltos" namespace in the management cluster to update the status of the associated `ClusterSummary` object, which then triggers a reconcile to 
re-deploy the spec to the target cluster.

> NOTE: The `ResourceSummary` and `ClusterSummary` are CRDs provided by Sveltos.

## Using Drift Ignore

Certain resources can be completely opted out of drift correction by using this feature.
In the following example, the "ingress-nginx/ingress-nginx-controller" deployment is ignored for drift correction on the target cluster.

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  . . .
spec:
  . . .
  serviceSpec:
    services:
      . . .
    priority: 100
    syncMode: ContinuousWithDriftDetection
    driftIgnore:
      - target:
        group: apps
        version: v1
        kind: Deployment
        name: ingress-nginx-controller
        namespace: ingress-nginx
  . . .
```

If we manually remove the `app.kubernetes.io/managed-by=Helm` label, we can observe that the drift is not corrected as can be seen in the following watch output.

```sh
kubectl -n ingress-nginx get deployments.apps ingress-nginx-controller --show-labels -w
```
```console
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE     LABELS
ingress-nginx-controller   3/3     3            3           3h58m   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/managed-by=Helm,app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/part-of=ingress-nginx,app.kubernetes.io/version=1.11.0,helm.sh/chart=ingress-nginx-4.11.0
ingress-nginx-controller   3/3     3            3           3h59m   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/part-of=ingress-nginx,app.kubernetes.io/version=1.11.0,helm.sh/chart=ingress-nginx-4.11.0
```

This can also be verified by observing that `ignoreForConfigurationDrift: true` for the targeted resource in the `ResourceSummary` spec on the target cluster.

```yaml
kind: ResourceSummary
metadata:
  . . .
spec:
  chartResources:
  - chartName: ingress-nginx
    group:
    . . .
    - group: apps
      ignoreForConfigurationDrift: true
      kind: Deployment
      name: ingress-nginx-controller
      namespace: ingress-nginx
      version: v1
    releaseName: ingress-nginx
    releaseNamespace: ingress-nginx
status:
  helmResourceHashes:
  . . .
```

Yet another way to check if a resource is being ignored for drift is by verifying that `projectsveltos.io/driftDetectionIgnore: ok` annotation has been applied to it as can be seen below.

```sh
kubectl -n ingress-nginx get deployments.apps ingress-nginx-controller -o=jsonpath='{.metadata.annotations}'
{"deployment.kubernetes.io/revision":"1","meta.helm.sh/release-name":"ingress-nginx","meta.helm.sh/release-namespace":"ingress-nginx","projectsveltos.io/driftDetectionIgnore":"ok"}%
```

### Removing Drift Ignore

The drift ignore setting can be removed by removing the `.spec.serviceSpec.driftIgnore`.

## Using Drift Exclusions

Certain fields of a resource can be excluded from drift detection using this feature.
In the following example, the `.spec.replicas` field of the "ingress-nginx/ingress-nginx-controller" deployment on the target cluster is excluded from drift detection.

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  . . .
spec:
  . . .
  serviceSpec:
    services:
      . . .
    priority: 100
    syncMode: ContinuousWithDriftDetection
    driftExclusions:
      - paths:
        - "/spec/replicas"
        target:
          kind: Deployment
          name: ingress-nginx-controller
          namespace: ingress-nginx
  . . .
```

If we manually edit the replicas to be 1, the number of replicas is not corrected back to 3 as is indicated by the following watch output.

```sh
kubectl -n ingress-nginx get deployments.apps ingress-nginx-controller -o=jsonpath='{.spec.replicas}' -w
3111
```

We can also verify that this is the case by observing that the `ResourceSummary` object has the following patch in its spec now.

```yaml
kind: ResourceSummary
metadata:
  . . .
spec:
  chartResources:
  - chartName: ingress-nginx
    group:
    . . .
    releaseName: ingress-nginx
    releaseNamespace: ingress-nginx
  patches:
  - patch: |-
      - op: remove
        path: /spec/replicas
    target:
      kind: Deployment
      name: ingress-nginx-controller
      namespace: ingress-nginx
status:
  helmResourceHashes:
  . . .
```

### Removing Drift Exclusion

The drift exclusion can be removed by removing the `.spec.serviceSpec.driftExclusion` field and re-triggering the drift correction by editing any field in the "ingress-nginx/ingress-nginx-controller" deployment. This will force a drift correction and since the drift exclusion has been removed, it will restore the deployment to it's original spec.