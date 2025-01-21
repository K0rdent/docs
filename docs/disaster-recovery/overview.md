# Disaster Recovery Feature Overview

This document provides an overview of the disaster recovery feature used to back up and restore
a `kcm` management cluster.

The feature leverages [`velero`](https://velero.io/) for backup management
on the backend and integrates with the `kcm` to ensure data persistence and recovery.

## Motivation

The primary goal of this feature is to provide a reliable and efficient way to back up and restore
`kcm` deployment in the event of a disaster that impacts the management cluster.
By utilizing `velero` as the backup provider, we can ensure consistent backups across
different cloud storage while maintaining the integrity of critical resources.

The main goal of the feature is to provide:

- Backup: the ability to backup all configuration objects **created and managed by the `kcm`**
  into an offsite location.
- Restore: the ability to create configuration objects from a specific Backup.
- Disaster Recovery: the ability to restore the `kcm` system on another management cluster
  using restore capability, plus ensuring that clusters are not recreated or lost.
- Rollback: the possibility to manually restore after a specific event, e.g. a failed upgrade
  of the `kcm`.

## Velero as Provider for Backups

[`Velero`](https://velero.io/) is an open-source tool that simplifies backing up and restoring clusters as well as individual resources. It seamlessly integrates into the `kcm` management environment to provide robust disaster recovery capabilities.

The `velero` instance is installed as part of the `kcm` and included in its helm chart,
hence the installation process might be fully customized.

The `kcm` manages the schedule and is responsible for collecting the sufficient
data to be included in a backup.

Customization options for the installation and backups are covered by [the according section](customization.md).

## Scheduled Backups

### Preparation

Before the creation of scheduled backups, several actions should be performed beforehand:

1. Prepare a cloud storage to save backups to (e.g. `Amazon S3`).
2. Create a [`BackupStorageLocation`](https://velero.io/docs/v1.15/api-types/backupstoragelocation/)
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

### Create Backup

To set a `ManagementBackup` object to create periodic backups,
set the `.spec.schedule` field with a [Cron](https://en.wikipedia.org/wiki/Cron) expression.
If the `.spec.schedule` is not set, the [backup on demand](#backup-on-demand) will be created instead.

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

## Backup on Demand

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

## What's Included in the Backup

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

> NOTE: Caveats and limitations
>
> Please refer to the
> [official migration documentation](https://velero.io/docs/v1.15/migration-case/#before-migrating-your-cluster)
> to be familiarized with potential limitations.

To restore from a backup in the event of a disaster, the following actions should be
performed:

1. Ensure a сlean `kcm` installation. For more information, please refer
   to the [installation guide](../usage/installation.md#extended-management-configuration).

    For example:

    ```bash
    helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm \
     --version <version> \
     --create-namespace \
     --namespace kcm-system \
     --set controller.createManagement=false \
     --set controller.createAccessManagement=false \
     --set controller.createRelease=false \
     --set controller.createTemplates=false
    ```

2. Create the `BackupStorageLocation`/`Secret` objects that had been
   created during the [preparation stage](#preparation) (preferably the same depending on a plugin).
3. Restore the `kcm` system: either use the [velero CLI](https://velero.io/docs/v1.15/basic-install/#install-the-cli)
   or directly create [`Restore`](https://velero.io/docs/v1.15/api-types/restore/) object.

    > EXAMPLE: A `Restore` object:
    >
    >```yaml
    > apiVersion: velero.io/v1
    > kind: Restore
    > metadata:
    >   name: <restore-name>
    >   namespace: kcm-system
    > spec:
    >   backupName: <backup-name>
    >   excludedResources:
    >   # the following are velero defaults, so it is recommended to keep them
    >   - nodes
    >   - events
    >   - events.events.k8s.io
    >   - backups.velero.io
    >   - restores.velero.io
    >   - resticrepositories.velero.io
    >   - csinodes.storage.k8s.io
    >   - volumeattachments.storage.k8s.io
    >   - backuprepositories.velero.io
    >   existingResourcePolicy: update
    >   includedNamespaces:
    >   - '*'
    >```

    > EXAMPLE: A `velero` CLI command:
    >
    >```bash
    > velero --namespace kcm-system restore create <restore-name> --existing-resource-policy update --from-backup <backup-name>
    >```

4. Wait until the `Restore` status is `Completed` and all `kcm` components are up and running.

<!-- ## Upgrades and rollback

TODO: fill out this section when the upgrade backup is implemented

Describe that the backup is created each time before the `kcm` upgrade.

Refer to how to rollback properly (probably just follow the restoration part)

TBD, no implementation exists yet -->

## Caveats / Limitation

<!-- TODO: not sure whether it is okay to mention that explicitly since we could implement
it somewhere in the future utilizing velero hooks -->

All `velero` caveats and limitations are transitively implied in the `k0rdent`.

In particular, that means no backup encryption is provided until it is implemented
by a `velero` plugin that supports both encryption and cloud storage backups.
