# Managed Cluster update

To update the `ManagedCluster`, update `.spec.template` in the `ManagedCluster`
object to the new `ClusterTemplate` name:

Run:

```shell
kubectl patch managedcluster.hmc <cluster-name> -n <namespace> --patch '{"spec":{"template":"<new-template-name>"}}' --type=merge
```

Then, check the status of the `ManagedCluster` object:

```shell
kubectl get managedcluster.hmc <cluster-name> -n <namespace>
```

In the commands above, replace the parameters enclosed in angle brackets with
the corresponding values.

To get more details, run the previous command with the `-o=yaml` option and
check the `.status.conditions`.

> NOTE:
> The `ManagedCluster` is allowed to be updated to specific templates only.
> The templates available for the update are defined in the
> `ClusterTemplateChain` objects. Also, the `AccessManagement` object should
> contain properly configured `spec.accessRules` with the list of
> `ClusterTemplateChain` object names and the namespaces where the supported
> templates from the chain spec will be delivered. For details, see:
> [Template Life Cycle Management](../template/main.md/#template-life-cycle-management).

<!---
TODO: Later all `ClusterTemplates` that are available for the update will be shown in the `ManagedCluster` status.
-->
