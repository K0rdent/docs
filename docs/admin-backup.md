# Backing up and Restoring a k0rdent Management Cluster

Any production system needs to provide Disaster Recovery features, and the heart of these capabilities is the ability to
perform backup and restore operations. In this chapter we'll look at backing up and restoring k0rdent management cluster
so that in case of an emergency, you can restore your system to its previous condition.

While it's possible to back up a Kubernetes cluster manually, it's better to build on the work
of others. In this case we're going to leverage the [`velero`](https://velero.io/) project for backup management
on the backend and see how it integrates with k0rdent to ensure data persistence and recovery.

## Motivation

The primary goal of this feature is to provide a reliable and efficient way to back up and restore
a k0rdent deployment in the event of a disaster that impacts the management cluster.
By using `velero` as the backup provider, we can create consistent backups across
different cloud storage while maintaining the integrity of critical resources.

The main goal of the feature is to provide:

- Management Backup: The ability to backup all configuration objects created and managed by k0rdent, including
  into an offsite location.
- Restore: The ability to create configuration objects from a specific Management Backup in order to create a management
  cluster in the same state that existed at the time of backup without (re)provisioning of cloud resources.
- Disaster Recovery: The ability to restore k0rdent on another management cluster
  using restore capability, plus ensuring that clusters are not recreated or lost.
- Rollback: the possibility to manually restore after a specific event, such as a failed k0rdent upgrade

## Velero as Provider for Management Backups

[`Velero`](https://velero.io/) is an open-source tool that simplifies backing up and restoring clusters as well as individual resources. It seamlessly integrates into k0rdent management environment to provide robust disaster recovery capabilities.

The `velero` instance is part of the Helm chart that installs k0rdent, which means that it
can be customized if necessary. (You can find information on customizing Velero [below](#customization))

k0rdent manages the schedule and is responsible for collecting data to be included in a backup.

## Scheduled Management Backups

Backups should be scheduled on a regular basis, depending on how often information changes.

### Preparation

Before you create a scheduled backup, you need to perform a few preparatory steps:

  1. If no `velero` plugins have been installed as suggested
     in the [corresponding section](#customization),
     install it by modifying the `Management` object:

      ```yaml
      apiVersion: k0rdent.mirantis.com/v1alpha1
      kind: Management
      metadata:
        name: kcm
      spec:
        # ... 
        core:
          kcm:
            config:
              velero:
                initContainers:
                - name: velero-plugin-for-<provider-name>
                  image: velero/velero-plugin-for-<provider-name>:<provider-plugin-tag>
                  imagePullPolicy: IfNotPresent
                  volumeMounts:
                  - mountPath: /target
                    name: plugins
        # ...
      ```

 2. Prepare a cloud storage location, such as an Amazon S3 bucket, to which to save backups.

 3. Create a [`BackupStorageLocation`](https://velero.io/docs/v1.15/api-types/backupstoragelocation/)
    object referencing a `Secret` with credentials to access the cloud storage
    (if the multiple credentials feature is supported by the plugin).

    For example, if you are using Amazon S3, your `BackupStorageLocation` and the related `Secret` might look like this:

    ```yaml
    ---
    apiVersion: v1
    data:
      # base64-encoded credentials for Amazon S3 in the following format:
      # [default]
      # aws_access_key_id = EXAMPLE_ACCESS_KEY_ID
      # aws_secret_access_key = EXAMPLE_SECRET_ACCESS_KEY
           cloud: W2RlZmF1bHRdCmF3c19hY2Nlc3Nfa2V5X2lkID0gRVhBTVBMRV9BQ0NFU1NfS0VZX0lECmF3c19zZWNyZXRfYWNjZXNzX2tleSA9IEVYQU1QTEVfU0VDUkVUX0FDQ0VTU19LRVkKICA=
    kind: Secret
    metadata:
      name: cloud-credentials
      namespace: kcm-system
    type: Opaque
    ---
    apiVersion: velero.io/v1
    kind: BackupStorageLocation
    metadata:
      name: aws-s3
      namespace: kcm-system
    spec:
      config:
        region: <your-region-name>
      default: true # optional, if not set, then storage location name must always be set in ManagementBackup
      objectStorage:
        bucket: <your-bucket-name>
      provider: aws
      backupSyncPeriod: 1m
      credential:
        name: cloud-credentials
        key: cloud
    ```

You can get more information how to build these objects at the [official Velero documentation](https://velero.io/docs/v1.15/locations).

### Create a Management Backup

Periodic backups are handled by a `ManagementBackup` object, which uses a [Cron](https://en.wikipedia.org/wiki/Cron) expression
for its `.spec.schedule` field. 
If you don't set the `.spec.schedule` field, Velero will instead work with  [backup on demand](#management-backup-on-demand).

Optionally, set the name of the `BackupStorageLocation` `.spec.backup.storageLocation`.
The default location is the `BackupStorageLocation` object with `.spec.default` set to `true`.

For example, you can create a `ManagementBackup` object that backs up to the storage object created in the previous step
every 6 minutes would look like this:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ManagementBackup
metadata:
  name: kcm
spec:
  schedule: "0 */6 * * *"
  storageLocation: aws-s3
```

## Management Backup on Demand

To create a single backup of the existing k0rdent management cluster information, you can create a `ManagementBackup` object
using a YAML document and the `kubectl` CLI. The object then creates only one instance of a backup. For example you can backup
to the location created earlier:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ManagementBackup
metadata:
  name: example-backup
spec:
  storageLocation: aws-s3
```

## What's Included in the Management Backup

The backup includes all of k0rdent `kcm` component resources, parts of the `cert-manager`
components required for other components creation, and all the required resources
of `CAPI` and the `ClusterDeployment`s currently in use in the management cluster.

> EXAMPLE: An example set of labels, and objects satisfying these labels will
> be included in the backup:
>
> ```text
> cluster.x-k8s.io/cluster-name="cluster-deployment-name"
> cluster.x-k8s.io/provider="bootstrap-k0sproject-k0smotron"
> cluster.x-k8s.io/provider="cluster-api"
> cluster.x-k8s.io/provider="control-plane-k0sproject-k0smotron"
> cluster.x-k8s.io/provider="infrastructure-aws"
> controller.cert-manager.io/fao="true"
> helm.toolkit.fluxcd.io/name="cluster-deployment-name"
> k0rdent.mirantis.com/component="kcm"
> ```

## Restoration

> NOTE: 
> Please refer to the
> [official migration documentation](https://velero.io/docs/v1.15/migration-case/#before-migrating-your-cluster)
> to familiarize yourself with potential limitations of the Velero backup system.

In the event of disaster, you can restore from a backup by doing the following:

1. Create a clean k0rdent installation, including `velero` and [its plugins](#customization). Specifically, you want to **avoid** creating a `Management` object
   and similar objects because they will be part of your restored cluster. You can remove these objects after
   installation, but you can also install k0rdent without them in the first place:

    ```bash
    helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm \
     --version <version> \
     --create-namespace \
     --namespace kcm-system \
     --set controller.createManagement=false \
     --set controller.createAccessManagement=false \
     --set controller.createRelease=false \
     --set controller.createTemplates=false \
     --set velero.initContainers[0].name=velero-plugin-for-<provider-name> \
     --set velero.initContainers[0].image=velero/velero-plugin-for-<provider-name>:<provider-plugin-tag> \
     --set velero.initContainers[0].volumeMounts[0].mountPath=/target \
     --set velero.initContainers[0].volumeMounts[0].name=plugins
    ```

2. Create the `BackupStorageLocation`/`Secret` objects that were
   created during the [preparation stage](#preparation) of creating a backup (preferably the same depending on a plugin).
1. Restore the `kcm` system creating the [`Restore`](https://velero.io/docs/v1.15/api-types/restore/) object,
   it is important to set the `.spec.existingResourcePolicy` field value to `update`:

    ```yaml
     apiVersion: velero.io/v1
     kind: Restore
     metadata:
       name: <restore-name>
       namespace: kcm-system
     spec:
       backupName: <backup-name>
       existingResourcePolicy: update
       includedNamespaces:
       - '*'
    ```

1. Wait until the `Restore` status is `Completed` and all `kcm` components are up and running.

### Caveats

For some CAPI providers it is necessary to make changes to the `Restore`
object due to the large number of different resources and logic in each provider.
The resources described below are not excluded from a `ManagementBackup` by
default to avoid logical dependencies on one or another provider
and continue to be provider-agnostic.

> NOTE:
> The described caveats apply only to the step with the `Restore`
> object creation and do not affect the other steps.

#### Azure (CAPZ)

The following resources should be excluded from the `Restore` object:

- `natgateways.network.azure.com`
- `resourcegroups.resources.azure.com`
- `virtualnetworks.network.azure.com`
- `virtualnetworkssubnets.network.azure.com`

Due to the [webhook conversion](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion),
objects of these resources cannot be restored, and they will
be created in the management cluster by the `CAPZ` provider
automatically with the same `spec` as in the backup.

The resulting `Restore` object:

```yaml
 apiVersion: velero.io/v1
 kind: Restore
 metadata:
   name: <restore-name>
   namespace: kcm-system
 spec:
   backupName: <backup-name>
   existingResourcePolicy: update
   excludedResources:
   - natgateways.network.azure.com
   - resourcegroups.resources.azure.com
   - virtualnetworks.network.azure.com
   - virtualnetworkssubnets.network.azure.com
   includedNamespaces:
   - '*'
```

#### vSphere (CAPV)

The following resources should be excluded from the `Restore` object:

- `mutatingwebhookconfiguration.admissionregistration.k8s.io`
- `validatingwebhookconfiguration.admissionregistration.k8s.io`

Due to the [Velero Restoration Order](https://velero.io/docs/v1.15/restore-reference/#restore-order),
some of the `CAPV` core objects cannot be restored,
and they will not be restored automatically.
Since all of the objects have already passed both mutations
and validations, there is not much sense in validating them again.
The webhook configurations will be restored during installation
of the `CAPV` provider.

The resulting `Restore` object:

```yaml
 apiVersion: velero.io/v1
 kind: Restore
 metadata:
   name: <restore-name>
   namespace: kcm-system
 spec:
   backupName: <backup-name>
   existingResourcePolicy: update
   excludedResources:
   - mutatingwebhookconfiguration.admissionregistration.k8s.io
   - validatingwebhookconfiguration.admissionregistration.k8s.io
   includedNamespaces:
   - '*'
```

## Upgrades and rollbacks

The Disaster Recovery Feature provides a way to create backups
on each `kcm` upgrade automatically.

### Automatic Management Backups

Each `ManagementBackup` with *non-empty* `.spec.schedule` field
can enable the automatic creation of backups before
[upgrading](admin-upgrading-k0rdent.md) to a new version.

To enable, set the `.spec.performOnManagementUpgrade` to `true`.

> EXAMPLE: An example of a `ManagementBackup` object with enabled auto-backup before the `kcm` version upgrade:
>
>```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ManagementBackup
> metadata:
>   name: example-backup
> spec:
>   schedule: "0 */6 * * *"
>   performOnManagementUpgrade: true
>```

After the enablement, before each upgrade of `kcm` to a new version,
a new backup will be created.

Automatically created backups have the
following name template to make it easier to find them:
the name of the `ManagementBackup` object with enabled `performOnManagementUpgrade`
concatenates with the name of the release before the upgrade,
e.g. `example-backup-kcm-0-1-0`.

Automatically created backups have the following label
`k0rdent.mirantis.com/release-backup`
with the name of the release before the upgrade as its value
to simplify querying if required.

### Rollbacks

If during the `kcm` upgrade a failure happens, a rollback operation
should be performed to restore the `kcm` to its before-the-upgrade state:

1. Follow the first 2 steps from the [restoration section](#restoration),
   creating a clean `kcm` installation and `BackupStorageLocation`/`Secret`.
> WARNING:
> Please consider the [restoration caveats](#caveats) section before proceeding.

1.  Create the `ConfigMap` object with patches to revert the `Management`
    `.spec.release`, substitute the `<version-before-upgrade>` with
    the version of `kcm` before the upgrade, and create the `Restore` object
    propagating the `ConfigMap` to it:

     ```yaml
     ---
     apiVersion: v1
     data:
       patch-mgmt-spec-release: |
         version: v1
         resourceModifierRules:
         - conditions:
             groupResource: managements.k0rdent.mirantis.com
           patches:
           - operation: replace
             path: "/spec/release"
             value: "<version-before-upgrade>"
     kind: ConfigMap
     metadata:
       name: patch-mgmt-spec-release
       namespace: kcm-system
     ---
     apiVersion: velero.io/v1
     kind: Restore
     metadata:
       name: <restore-name>
       namespace: kcm-system
     spec:
       backupName: <backup-name>
       existingResourcePolicy: update
       includedNamespaces:
       - '*'
       resourceModifier: # propagate patches
         kind: ConfigMap
         name: patch-mgmt-spec-release
     ```

1. Wait until the `Restore` status is `Completed` and all `kcm` components are up and running.
1. Optionally delete the created `ConfigMap`.

## Caveats / Limitations

The credentials stored in backups might and will get stale,
so a proper rotation should be considered beforehand.

All `velero` caveats and limitations are transitively implied in the `k0rdent`.

In particular, that means no backup encryption is provided until it is implemented
by a `velero` plugin that supports encryption and cloud storage backups.

## Velero Backups / Restores deletion

### Delete Restores

To delete `velero` `Restore` from the management cluster
**and** from the cloud storage, delete `restores.velero.io` object(s),
e.g. with the following command:

```bash
kubectl delete restores.velero.io -n kcm-system <restore-name>
```

> WARNING:
> Deletion of a `Restore` object deletes it from both
> the management cluster and from the cloud storage.

### Delete Backups

To remove `velero` `Backup` from the management cluster,
delete `backups.velero.io` object(s), e.g. with the following command:

```bash
kubectl delete backups.velero.io -n kcm-system <velero-backup-name>
```

> HINT:
> The command above only removes objects from
> the cluster, the data continues to persist
> on the cloud storage.
>
> The deleted object will be recreated in the
> cluster if its `BackupStorageLocation` `.spec.backupSyncPeriod`
> is set and does not equal `0`.

To delete `velero` `Backup` from the management cluster
**and** from the cloud storage, create the following `DeleteBackupRequest` object:

```yaml
apiVersion: velero.io/v1
kind: DeleteBackupRequest
metadata:
  name: delete-backup-completely
  namespace: kcm-system
spec:
  backupName: <velero-backup>
```

> WARNING:
> Deletion of a `Backup` object via the `DeleteBackupRequest`
> deletes it from both
> the management cluster and from the cloud storage.

Optionally, delete the created `DeleteBackupRequest` object
from the cluster after `Backup` has been deleted.

For reference, follow the [official documentation](https://velero.io/docs/v1.15/backup-reference/#deleting-backups).

# Customization

This section covers different topics of customization regarding backing up and restoring k0rdent.

## Velero installation

The Velero helm chart is supplied with the
[k0rdent helm chart](https://vmware-tanzu.github.io/helm-charts/)
and is enabled by default. There are 2 ways of customizing the chart values:

1. Install using `helm` and add corresponding parameters to the `helm install` command.

    > NOTE:
    Only a plugin is required during the restoration, the other parameters
    are optional to be set.
    
    > EXAMPLE: An example of `helm install` with a configured plugin, `BackupStorageLocation`
    > and propagated credentials:
    ```shell
    helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm \
     --version <version> \
     --create-namespace \
     --namespace kcm-system \
     --set-file velero.credentials.secretContents.cloud=<FULL PATH TO FILE> \
     --set velero.credentials.useSecret=true \
     --set velero.backupsEnabled=true \
     --set velero.configuration.backupStorageLocation[0].name=<backup-storage-location-name> \
     --set velero.configuration.backupStorageLocation[0].provider=<provider-name> \
     --set velero.configuration.backupStorageLocation[0].bucket=<bucket-name> \
     --set velero.configuration.backupStorageLocation[0].config.region=<region> \
     --set velero.initContainers[0].name=velero-plugin-for-<provider-name> \
     --set velero.initContainers[0].image=velero/velero-plugin-for-<provider-name>:<provider-plugin-tag> \
     --set velero.initContainers[0].volumeMounts[0].mountPath=/target \
     --set velero.initContainers[0].volumeMounts[0].name=plugins
    ```

2. Create or modify the existing `Management` object in the `.spec.config.kcm`.
    > NOTE:
    > Only a plugin is required during the restoration, the other parameters
    > are optional to be set.
    
    > EXAMPLE: An example of a `Management` object with a configured plugin and enabled metrics:
    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: Management
    metadata:
      name: kcm
    spec:
      # ...
      core:
        kcm:
          config:
            velero:
              initContainers:
              - name: velero-plugin-for-<provider-name>
                image: velero/velero-plugin-for-<provider-name>:<provider-plugin-tag>
                imagePullPolicy: IfNotPresent
                volumeMounts:
                - mountPath: /target
                  name: plugins
              metrics:
                enabled: true
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

## Putting extra objects in a Management Backup

If you need to back up objects other than those [backed up by default](admin-backup.md#whats-included-in-the-management-backup),
you can add the label `k0rdent.mirantis.com/component="kcm"` to these objects.

All objects containing the label will be automatically added to the management backup.