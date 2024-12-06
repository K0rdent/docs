# Templates system

By default, 2A delivers a set of default `ProviderTemplate`, `ClusterTemplate` and `ServiceTemplate` objects:

* `ProviderTemplate`
   The template containing the configuration of the provider (ie k0smotron). Cluster-scoped.
* `ClusterTemplate`
   The template containing the configuration of the cluster objects. Namespace-scoped.
* `ServiceTemplate`
   The template containing the configuration of the service to be installed on the managed cluster. Namespace-scoped.

All Templates are immutable. You can also build your own templates and use them for deployment along with the
templates shipped with 2A.

## Template Life Cycle Management

Cluster and Service Templates can be delivered to target namespaces using the `AccessManagement`,
`ClusterTemplateChain` and `ServiceTemplateChain` objects. `AccessManagement` object contains the list of
access rules to apply. Each access rule contains the namespaces' definition to deliver templates into and
the template chains. Each `ClusterTemplateChain` and `ServiceTemplateChain` contains the supported templates
and the upgrade sequences for them.

The example of the Cluster Template Management:

1. Create `ClusterTemplateChain` object in the system namespace (defaults to `hmc-system`). Properly configure
   the list of `.spec.supportedTemplates[].availableUpgrades` for the specified `ClusterTemplate` if the upgrade is allowed. For example:

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: ClusterTemplateChain
metadata:
  name: aws
  namespace: hmc-system
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

The HMC controllers will deliver all the `ClusterTemplate` objects across the target namespaces.
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
