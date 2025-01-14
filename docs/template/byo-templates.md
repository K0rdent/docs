# Bring your own Templates

This guide outlines the steps to bring your own Template to kcm.

## Create a Source Object

> INFO:
> Skip this step if you're using an existing source.

A source object defines where the Helm chart is stored. The source can be one of the following types:
[HelmRepository](https://fluxcd.io/flux/components/source/helmrepositories/),
[GitRepository](https://fluxcd.io/flux/components/source/gitrepositories/) or
[Bucket](https://fluxcd.io/flux/components/source/buckets/).

> NOTES:
> 1. The source object must exist in the same namespace as the Template.
> 2. For cluster-scoped `ProviderTemplates`, the referenced source must reside in the **system** namespace
> (default: `kcm-system`).

### Example: Custom Source Object with HelmRepository Kind

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: custom-templates-repo
  namespace: kcm-system
spec:
  insecure: true
  interval: 10m0s
  provider: generic
  type: oci
  url: oci://ghcr.io/external-templates-repo/charts
```

## Create the Template

Create a Template object of the desired type:

* `ClusterTemplate`
* `ServiceTemplate`
* `ProviderTemplate`

For `ClusterTemplate` and `ServiceTemplate` configure the namespace where this template should reside
(`metadata.namespace`).
The custom Template requires a helm chart definition in the `.spec.helm.chartSpec` field of the
[HelmChartSpec](https://fluxcd.io/flux/components/source/api/v1/#source.toolkit.fluxcd.io/v1.HelmChartSpec) kind or
the reference to already existing `HelmChart` object in `.spec.helm.chartRef`.

> NOTE:
> `spec.helm.chartSpec` and `spec.helm.chartRef` are mutually exclusive.

To automatically create the `HelmChart` for the `Template`, configure the following custom helm chart parameters
under `spec.helm.chartSpec`:

| **Field**                                                                                                                                                   | **Description**                                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| `sourceRef`<br/>[LocalHelmChartSourceReference](https://fluxcd.io/flux/components/source/api/v1/#source.toolkit.fluxcd.io/v1.LocalHelmChartSourceReference) | Reference to the source object (e.g., `HelmRepository`, `GitRepository`, or `Bucket`) in the same namespace as the Template. |
| `chart`<br/>string                                                                                                                                          | The name of the Helm chart available in the source.                                                                          |
| `version`<br/>string                                                                                                                                        | Version is the chart version semver expression. Defaults to **latest** when omitted.                                         |
| `interval`<br/>[Kubernetes meta/v1.Duration](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#Duration)                                              | The frequency at which the `sourceRef` is checked for updates. Defaults to **10 minutes**.                                   |

For the complete list of the `HelmChart` parameters, see:
[HelmChartSpec](https://fluxcd.io/flux/components/source/api/v1/#source.toolkit.fluxcd.io/v1.HelmChartSpec).

The controller will automatically create the `HelmChart` object based on the chartSpec defined in
`.spec.helm.chartSpec`.

> NOTE:
> `ClusterTemplate` and `ServiceTemplate` objects should reside in the same namespace as the `ClusterDeployment`
> referencing them. The `ClusterDeployment` can't reference the Template from another namespace (the creation request will
> be declined by the admission webhook). All `ClusterTemplates` and `ServiceTemplates` shipped with kcm reside in the
> system namespace (defaults to `kcm-system`). To get the instructions on how to distribute Templates along multiple
> namespaces, read [Template Life Cycle Management](main.md#template-life-cycle-management).

### Example: Custom ClusterTemplate with the Chart Definition to Create a new HelmChart

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterTemplate
metadata:
  name: custom-template
  namespace: kcm-system
spec:
  providers:
    - bootstrap-k0smotron
    - control-plane-k0smotron
    - infrastructure-openstack
  helm:
    chartSpec:
      chart: os-k0smotron
      sourceRef:
        kind: HelmRepository
        name: custom-templates-repo
```

### Example: Custom ClusterTemplate Referencing an Existing HelmChart object

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterTemplate
metadata:
  name: custom-template
  namespace: kcm-system
spec:
  helm:
    chartRef:
      kind: HelmChart
      name: custom-chart
```


## Required and exposed providers definition

The `*Template` object must specify the list of Cluster API providers that are either **required** (for
`ClusterTemplates` and `ServiceTemplates`) or **exposed** (for `ProviderTemplates`). These providers include
`infrastructure`, `bootstrap`, and `control-plane`. This can be achieved in two ways:

1. By listing the providers explicitly in the `spec.providers` field.
2. Alternatively, by including specific annotations in the `Chart.yaml` of the referenced Helm chart.
The annotations should list the providers as a `comma-separated` value.

For example:

`Template` spec:

```yaml
spec:
  providers:
  - bootstrap-k0smotron
  - control-plane-k0smotron
  - infrastructure-aws
```

`Chart.yaml`:

```bash
annotations:
  cluster.x-k8s.io/provider: infrastructure-aws, control-plane-k0smotron, bootstrap-k0smotron
```

## Compatibility attributes

Each of the `*Template` resources has compatibility versions attributes to constraint the core `CAPI`, `CAPI` provider or Kubernetes versions.
CAPI-related version constraints must be set in the [`CAPI` contract format](https://cluster-api.sigs.k8s.io/developer/providers/contracts/overview).
Kubernetes version constraints must be set in the Semantic Version format.
Each attribute can be set either via the corresponding `.spec` fields or via the annotations.
Values set via the `.spec` have precedence over the values set via the annotations.

> NOTE:
> All of the compatibility attributes are optional, and validation checks only take place
> if **both** of the corresponding type attributes
> (e.g. provider contract versions in both `ProviderTemplate` and `ClusterTemplate`) are set.

1. The `ProviderTemplate` resource has dedicated fields to set compatible `CAPI` contract versions along
with CRDs contract versions supported by the provider.
Given contract versions will be then set accordingly in the `.status` field.
Compatibility contract versions are key-value pairs, where the key is **the core `CAPI` contract version**,
and the value is an underscore-delimited (_) list of provider contract versions supported by the core `CAPI`.
For the core `CAPI` Template values should be empty.

    Example with the `.spec`:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ProviderTemplate
    # ...
    spec:
      providers:
      - infrastructure-aws
      capiContracts:
        # commented is the example exclusively for the core CAPI Template
        # v1alpha3: ""
        # v1alpha4: ""
        # v1beta1: ""
        v1alpha3: v1alpha3
        v1alpha4: v1alpha4
        v1beta1: v1beta1_v1beta2
    ```

    Example with the `annotations` in the `Chart.yaml` with the same logic
    as in the `.spec`:

    ```yaml
    annotations:
      cluster.x-k8s.io/provider: infrastructure-aws
      cluster.x-k8s.io/v1alpha3: v1alpha3
      cluster.x-k8s.io/v1alpha4: v1alpha4
      cluster.x-k8s.io/v1beta1: v1beta1_v1beta2
    ```

1. The `ClusterTemplate` resource has dedicated fields to set an exact compatible Kubernetes version
in the Semantic Version format and required contract versions per each provider to match against
the related `ProviderTemplate` objects.
Given compatibility attributes will be then set accordingly in the `.status` field.
Compatibility contract versions are key-value pairs, where the key is **the name of the provider**,
and the value is the provider contract version required to be supported by the provider.

    Example with the `.spec`:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ClusterTemplate
    # ...
    spec:
      k8sVersion: 1.30.0 # only exact semantic version is applicable
      providers:
      - bootstrap-k0smotron
      - control-plane-k0smotron
      - infrastructure-aws
      providerContracts:
        bootstrap-k0smotron: v1beta1 # only a single contract version is applicable
        control-plane-k0smotron: v1beta1
        infrastructure-aws: v1beta2
    ```

    Example with the `.annotations` in the `Chart.yaml`:

    ```yaml
    annotations:
      cluster.x-k8s.io/provider: infrastructure-aws, control-plane-k0smotron, bootstrap-k0smotron
      cluster.x-k8s.io/bootstrap-k0smotron: v1beta1
      cluster.x-k8s.io/control-plane-k0smotron: v1beta1
      cluster.x-k8s.io/infrastructure-aws: v1beta2
      k0rdent.mirantis.com/k8s-version: 1.30.0
    ```

1. The `ServiceTemplate` resource has dedicated fields to set an compatibility constrained
Kubernetes version to match against the related `ClusterTemplate` objects.
Given compatibility values will be then set accordingly in the `.status` field.

    Example with the `.spec`:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: ServiceTemplate
    # ...
    spec:
      k8sConstraint: "^1.30.0" # only semantic version constraints are applicable
    ```

    Example with the `annotations` in the `Chart.yaml`:

    ```yaml
    k0rdent.mirantis.com/k8s-version-constraint: ^1.30.0
    ```

### Compatibility attributes enforcement

The aforedescribed attributes are being checked sticking to the following rules:

* both the exact and constraint version of the same type (e.g. `k8sVersion` and `k8sConstraint`) must
be set otherwise no check is performed;
* if a `ClusterTemplate` object's providers contract version does not satisfy contract versions
from the related `ProviderTemplate` object, the updates to the `ClusterDeployment` object will be blocked;
* if a `ProviderTemplate` object's `CAPI` contract version
(e.g. in a `v1beta1: v1beta1_v1beta2` key-value pair, the key `v1beta1` is the core `CAPI` contract version)
is not listed in the core `CAPI` `ProviderTemplate` object, the updates to the `Management` object will be blocked;
* if a `ClusterTemplate` object's exact kubernetes version does not satisfy the kubernetes version
constraint from the related `ServiceTemplate` object, the updates to the `ClusterDeployment` object will be blocked.

## Remove Templates shipped with kcm

If you need to limit the templates that exist in your kcm installation, follow the instructions below:

1. Get the list of `ProviderTemplates`, `ClusterTemplates` or `ServiceTemplates` shipped with kcm. For example,
for `ClusterTemplate` objects, run:

    ```bash
    kubectl get clustertemplates -n kcm-system -l helm.toolkit.fluxcd.io/name=kcm-templates
    ```

    Example output:

    ```bash
    NAME                       VALID
    aws-hosted-cp              true
    aws-standalone-cp          true
    ```

2. Remove the template from the list using `kubectl delete`. For example:

    ```bash
    kubectl delete clustertemplate -n kcm-system <template-name>
    ```
