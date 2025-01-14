# Credential Distribution System

The kcm system provides a mechanism to distribute `Credential` objects across namespaces using the
`AccessManagement` object. This object defines a set of `accessRules` that determine how credentials are distributed.

Each access rule specifies:

1. The target namespaces where credentials should be delivered.
2. A list of `Credential` names to distribute to those namespaces.

The kcm controller will copy the specified `Credential` objects from the **system** namespace to the target
namespaces based on the `accessRules` in the `AccessManagement` spec.

> INFO:
> Access rules can also include `Cluster` and `Service` TemplateChains (`clusterTemplateChains` and
> `serviceTemplateChains`) to distribute templates to target namespaces.
> For more details, read: [Template Life Cycle Management](../template/main.md#template-life-cycle-management).

## How to Configure Credential Distribution

To configure the distribution of `Credential` objects:

1. Edit the `AccessManagement` object.
2. Populate the `.spec.accessRules` field with the list of `Credential` names and the target namespaces.

Here’s an example configuration:

```yaml
spec:
  accessRules:
  - targetNamespaces:
      list:
        - dev
        - test
    credentials:
      - aws-demo
      - azure-demo
```

In this example, the `aws-demo` and `azure-demo` `Credential` objects will be distributed to the `dev` and `test`
namespaces.


