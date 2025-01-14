# Templates system

By default, k0rdent delivers a set of default `ProviderTemplate`, `ClusterTemplate` and `ServiceTemplate` objects:

* `ProviderTemplate`
   The template containing the configuration of the provider (e.g., k0smotron). Cluster-scoped.
* `ClusterTemplate`
   The template containing the configuration of the cluster objects. Namespace-scoped.
* `ServiceTemplate`
   The template containing the configuration of the service to be installed on the cluster deployment. Namespace-scoped.

All Templates are immutable. You can also build your own templates and use them for deployment along with the
templates shipped with k0rdent.

## Template Naming Convention

The templates can have any name. However, since they are immutable, we have adopted a naming convention that includes semver in the name, i.e., `template-<major>-<minor>-<patch>`. Below are some examples for each of the templates.

> EXAMPLE: An example of a `ProviderTemplate` with its status.
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ProviderTemplate
> metadata:
>   name: cluster-api-0-0-4
> spec:
>   helm:
>     chartSpec:
>       chart: cluster-api
>       interval: 10m0s
>       reconcileStrategy: ChartVersion
>       sourceRef:
>         kind: HelmRepository
>         name: kcm-templates
>       version: 0.0.4
> status:
>   capiContracts:
>     v1alpha3: ""
>     v1alpha4: ""
>     v1beta1: ""
>   chartRef:
>     kind: HelmChart
>     name: cluster-api-0-0-4
>     namespace: kcm-system
>   config:
>     airgap: false
>     config: {}
>     configSecret:
>       create: false
>       name: ""
>       namespace: ""
>   description: A Helm chart for Cluster API core components
>   observedGeneration: 1
>   valid: true
> ```

> EXAMPLE: An example of a `ClusterTemplate` with its status.
> ```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ClusterTemplate
> metadata:
>   name: aws-standalone-cp-0-0-3
>   namespace: kcm-system
> spec:
>   helm:
>     chartSpec:
>       chart: aws-standalone-cp
>       interval: 10m0s
>       reconcileStrategy: ChartVersion
>       sourceRef:
>         kind: HelmRepository
>         name: kcm-templates
>       version: 0.0.3
> status:
>   chartRef:
>     kind: HelmChart
>     name: aws-standalone-cp-0-0-3
>     namespace: kcm-system
>   config:
>     bastion:
>       allowedCIDRBlocks: []
>       ami: ""
>       disableIngressRules: false
>       enabled: false
>       instanceType: t2.micro
>     clusterIdentity:
>       kind: AWSClusterStaticIdentity
>       name: ""
>     clusterNetwork:
>       pods:
>         cidrBlocks:
>         - 10.244.0.0/16
>       services:
>         cidrBlocks:
>         - 10.96.0.0/12
>     controlPlane:
>       amiID: ""
>       iamInstanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
>       imageLookup:
>         baseOS: ""
>         format: amzn2-ami-hvm*-gp2
>         org: "137112412989"
>       instanceType: ""
>       rootVolumeSize: 8
>     controlPlaneNumber: 3
>     extensions:
>       chartRepository: ""
>       imageRepository: ""
>     k0s:
>       version: v1.31.1+k0s.1
>     publicIP: false
>     region: ""
>     sshKeyName: ""
>     worker:
>       amiID: ""
>       iamInstanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
>       imageLookup:
>         baseOS: ""
>         format: amzn2-ami-hvm*-gp2
>         org: "137112412989"
>       instanceType: ""
>       rootVolumeSize: 8
>     workersNumber: 2
>   description: 'An kcm template to deploy a k0s cluster on AWS with bootstrapped control
>     plane nodes. '
>   observedGeneration: 1
>   providerContracts:
>     bootstrap-k0smotron: v1beta1
>     control-plane-k0smotron: v1beta1
>     infrastructure-aws: v1beta2
>   providers:
>   - bootstrap-k0smotron
>   - control-plane-k0smotron
>   - infrastructure-aws
>   valid: true
> ```

> EXAMPLE: An example of a `ServiceTemplate` with its status.
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
> status:
>   chartRef:
>     kind: HelmChart
>     name: kyverno-3-2-6
>     namespace: kcm-system
>   description: A Helm chart to refer the official kyverno helm chart
>   observedGeneration: 1
>   valid: true
> ```

## Template Life Cycle Management

Cluster and Service Templates can be delivered to target namespaces using the `AccessManagement`,
`ClusterTemplateChain` and `ServiceTemplateChain` objects. `AccessManagement` object contains the list of
access rules to apply. Each access rule contains the namespaces' definition to deliver templates into and
the template chains. Each `ClusterTemplateChain` and `ServiceTemplateChain` contains the supported templates
and the upgrade sequences for them.

The example of the Cluster Template Management:

1. Create `ClusterTemplateChain` object in the system namespace (defaults to `kcm-system`). Properly configure
   the list of `.spec.supportedTemplates[].availableUpgrades` for the specified `ClusterTemplate` if the upgrade is allowed. For example:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterTemplateChain
metadata:
  name: aws
  namespace: kcm-system
spec:
  supportedTemplates:
    - name: aws-standalone-cp-0-0-1
      availableUpgrades:
        - name: aws-standalone-cp-0-0-2
    - name: aws-standalone-cp-0-0-2
```

2. Edit `AccessManagement` object and configure the `.spec.accessRules`.
   For example, to apply all templates and upgrade sequences defined in the `aws` `ClusterTemplateChain` to the
   `default` namespace, the following `accessRule` should be added:

```yaml
spec:
  accessRules:
  - targetNamespaces:
      list:
        - default
    clusterTemplateChains:
      - aws
```

The kcm controllers will deliver all the `ClusterTemplate` objects across the target namespaces.
As a result, the new objects should be created:

* `ClusterTemplateChain` `default/aws`
* `ClusterTemplate` `default/aws-standalone-cp-0-0-1`
* `ClusterTemplate` `default/aws-standalone-cp-0-0-2` (available for the upgrade from `aws-standalone-cp-0-0-1`)

> NOTE:
>
> 1. The target `ClusterTemplate` defined as the available for the upgrade should reference the same helm chart name
> as the source `ClusterTemplate`. Otherwise, after the upgrade is triggered, the cluster will be removed and then,
> recreated from scratch even if the objects in the helm chart are the same.
> 2. The target template should not affect immutable fields or any other incompatible internal objects upgrades,
> otherwise the upgrade will fail.
