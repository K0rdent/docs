# Customization

This section covers different topics of customization regarding
[the disaster recovery feature](overview.md).

## Velero installation

The `kcm` helm chart supplied with the
[`velero` helm chart](https://vmware-tanzu.github.io/helm-charts/)
and is enabled by default.
There are 2 ways of customizing the chart values similar to
the [installation guide](../usage/installation.md#extended-management-configuration):

1. If installing using `helm` add corresponding parameters to the `helm install` command,
   for example:

    ```bash
    helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm \
     --version <version> \
     --create-namespace \
     --namespace kcm-system \
     --set velero.initContainers[0].name=velero-plugin-for-<PROVIDER NAME> \
     --set velero.initContainers[0].image=velero/velero-plugin-for-<PROVIDER NAME>:<PROVIDER PLUGIN TAG> \
     --set velero.initContainers[0].volumeMounts[0].mountPath=/target \
     --set velero.initContainers[0].volumeMounts[0].name=plugins
    ```

2. Create or modify the existing `Management` object in the `.spec.config.kcm`, for example:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: Management
    metadata:
      name: kcm
    spec:
      core:
        kcm:
          config:
            velero:
             initContainers:
               - name: velero-plugin-for-<PROVIDER NAME>
                 image: velero/velero-plugin-for-<PROVIDER NAME>:<PROVIDER PLUGIN TAG>
                 imagePullPolicy: IfNotPresent
                 volumeMounts:
                   - mountPath: /target
                     name: plugins
          # ...
    ```

To fully disable `velero`, set the `velero.enabled` parameter to `false`.

## Schedule expression format

The `ManagementBackup` `.spec.schedule` field accepts a correct
[Cron](https://en.wikipedia.org/wiki/Cron) expression,
along with the
[nonstandard predefined scheduling definitions](https://en.wikipedia.org/wiki/Cron#Nonstandard_predefined_scheduling_definitions)
and an extra definition `@every` with a number and a valid time unit
(valid time units are `ns, us (or µs), ms, s, m, h`).

The following list contains `.spec.schedule` acceptable example values:

- `0 */1 * * *` (standard Cron expression)
- `@hourly` (nonstandard predefined definition)
- `@every 1h` (extra definition)

## Putting extra objects in a Backup

If a situation arises in which it is necessary to back up some
additional objects in addition to those [backed up by default](overview.md#whats-included-in-the-backup),
it is enough to add the following label `k0rdent.mirantis.com/component="kcm"` to these objects.

All objects containing the label will be automatically added to the backup.
