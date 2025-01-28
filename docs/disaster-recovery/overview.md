# Disaster Recovery Feature Overview

This document provides an overview of the disaster recovery feature used to back up and restore
the **`kcm` management cluster**.

The feature leverages [`velero`](https://velero.io/) for backup management
on the backend and integrates with the `kcm` to ensure data persistence and recovery.

## Motivation

The primary goal of this feature is to provide a reliable and efficient way to back up and restore
`kcm` deployment in the event of a disaster that impacts the management cluster.
By utilizing `velero` as the backup provider, we can ensure consistent backups across
different cloud storage while maintaining the integrity of critical resources.

The main goal of the feature is to provide:

- Management Backup: the ability to backup all configuration objects **created and managed by the `kcm`**
  into an offsite location.
- Restore: the ability to create configuration objects from a specific Management Backup
  **without (re)provisioning of cloud resources**.
- Disaster Recovery: the ability to restore the `kcm` system on another management cluster
  using restore capability, plus ensuring that clusters are not recreated or lost.
- Rollback: the possibility to manually restore after a specific event, e.g. a failed upgrade
  of the `kcm`.

## Velero as Provider for Management Backups

[`Velero`](https://velero.io/) is an open-source tool that simplifies backing up and restoring clusters as well as individual resources. It seamlessly integrates into the `kcm` management environment to provide robust disaster recovery capabilities.

The `velero` instance is installed as part of the `kcm` and included in its helm chart,
hence the installation process might be fully customized.

The `kcm` manages the schedule and is responsible for collecting the sufficient
data to be included in a backup.

Customization options for the installation and backups are covered by [the according section](customization.md).

## Scheduled Management Backups

### Preparation

Before the creation of scheduled backups, several actions should be performed beforehand:

1. If no `velero` pluguns have not been yet installed as suggested
   in the [corresponding section](customization.md#velero-installation),
   install it modifying the `Management` object:

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

1. Prepare a cloud storage to save backups to (e.g. `Amazon S3`).
1. Create a [`BackupStorageLocation`](https://velero.io/docs/v1.15/api-types/backupstoragelocation/)
   object referencing a `Secret` with credentials to access the cloud storage
   (if the multiple credentials feature is supported by the plugin).

> EXAMPLE: An example of a `BackupStorageLocation` and the related `Secret` for the `Amazon S3`
> and the `AWS` provider:
>
>```yaml
> ---
> # Secret with the cloud storage credentials
> apiVersion: v1
> data:
>   # base64-encoded credentials, for the Amazon S3 in the following format:
>   # [default]
>   # aws_access_key_id = <AWS_ACCESS_KEY>
>   # aws_secret_access_key = <AWS_SECRET_ACCESS_KEY>
>   cloud: W2RlZmF1bHRdCmF3c19hY2Nlc3Nfa2V5X2lkID0gPEFXU19BQ0NFU1NfS0VZPgphd3Nfc2VjcmV0X2FjY2Vzc19rZXkgPSA8QVdTX1NFQ1JFVF9BQ0NFU1NfS0VZPgo=
> kind: Secret
> metadata:
>   name: cloud-credentials
>   namespace: kcm-system
> type: Opaque
> ---
> # Velero storage location
> apiVersion: velero.io/v1
> kind: BackupStorageLocation
> metadata:
>   name: aws-s3
>   namespace: kcm-system
> spec:
>   config:
>     region: <your-region-name>
>   default: true # optional, if not set, then storage location name must always be set in ManagementBackup
>   objectStorage:
>     bucket: <your-bucket-name>
>   provider: aws
>   backupSyncPeriod: 1m
>   credential:
>     name: cloud-credentials
>     key: cloud
>```

> HINT:
> For more comprehensive examples and to familiarize yourself with limitations and caveats
> please follow the link to the [official location documentation](https://velero.io/docs/v1.15/locations).

### Create Management Backup

To set a `ManagementBackup` object to create periodic backups,
set the `.spec.schedule` field with a [Cron](https://en.wikipedia.org/wiki/Cron) expression.
If the `.spec.schedule` is not set, the [backup on demand](#management-backup-on-demand) will be created instead.

Optionally, set the name of the `BackupStorageLocation` `.spec.backup.storageLocation`.
The default location is the `BackupStorageLocation` object with `.spec.default` set to `true`.

> EXAMPLE: An example of the `ManagementBackup` object with a schedule and
> the storage location:
>
>```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ManagementBackup
> metadata:
>   name: kcm
> spec:
>   schedule: "0 */6 * * *"
>   storageLocation: aws-s3
>```

## Management Backup on Demand

To create a single backup of the `kcm`, a `ManagementBackup` object can be created
manually, e.g. via the `kubectl` CLI. The object then creates only one instance of backup.

> EXAMPLE: An example of a `ManagementBackup` object:
>
>```yaml
> apiVersion: k0rdent.mirantis.com/v1alpha1
> kind: ManagementBackup
> metadata:
>   name: example-backup
> spec:
>   storageLocation: my-location
>```

## What's Included in the Management Backup

The backup includes all of the `kcm` component resources, parts of the `cert-manager`
components required for other components creation, and all the required resources
of `CAPI` and `ClusterDeployment`s currently in use in the management cluster.

> EXAMPLE: An example set of labels, and objects satisfying these labels will
> be included in the backup:
>
>```text
> cluster.x-k8s.io/cluster-name="cluster-deployment-name"
> cluster.x-k8s.io/provider="bootstrap-k0sproject-k0smotron"
> cluster.x-k8s.io/provider="cluster-api"
> cluster.x-k8s.io/provider="control-plane-k0sproject-k0smotron"
> cluster.x-k8s.io/provider="infrastructure-aws"
> controller.cert-manager.io/fao="true"
> helm.toolkit.fluxcd.io/name="cluster-deployment-name"
> k0rdent.mirantis.com/component="kcm"
>```

## Restoration

> NOTE:
>
> Please refer to the
> [official migration documentation](https://velero.io/docs/v1.15/migration-case/#before-migrating-your-cluster)
> to be familiarized with potential Velero limitations.

### General case

To restore from a backup in the event of a disaster, the following actions should be
performed:

1. Ensure a сlean `kcm` installation including `velero` and [its plugins](customization.md#velero-installation):

    ```bash
    helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm \
     --version <version> \
     --create-namespace \
     --namespace kcm-system \
     --set controller.createManagement=false \
     --set controller.createAccessManagement=false \
     --set controller.createRelease=false \
     --set controller.createTemplates=false
     --set velero.initContainers[0].name=velero-plugin-for-<provider-name> \
     --set velero.initContainers[0].image=velero/velero-plugin-for-<provider-name>:<provider-plugin-tag> \
     --set velero.initContainers[0].volumeMounts[0].mountPath=/target \
     --set velero.initContainers[0].volumeMounts[0].name=plugins
    ```

    For more information, please refer
    to the [installation guide](../usage/installation.md#extended-management-configuration).

1. Create the `BackupStorageLocation`/`Secret` objects that had been
   created during the [preparation stage](#preparation) (preferably the same depending on a plugin).
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

For some `CAPI` providers it is necessary to make changes to the `Restore`
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

## Upgrades and rollback

The Disaster Recovery Feature provides a way to create backups
on each `kcm` upgrade automatically.

### Automatic Management Backups

Each `ManagementBackup` with *non-empty* `.spec.schedule` field
can enable the automatic creation of backups before
[upgrading](../usage/upgrade.md) to a new version.

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

### Rollback

If during the `kcm` upgrade a failure happens, a rollback operation
should be performed to restore the `kcm` to its before-the-upgrade state:

1. Follow the first 2 steps from the [restoration section](#general-case),
   creating a clean `kcm` installation and `BackupStorageLocation`/`Secret`.

1. > WARNING:
   > Please consider the [restoration caveats](#caveats) section before proceeding.

    Create the `ConfigMap` object with patches to revert the `Management`
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

## Caveats / Limitation

<!-- TODO: not sure whether it is okay to mention that explicitly since we could implement
it somewhere in the future utilizing velero hooks -->

The credentials stored in backups might and will stale,
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
