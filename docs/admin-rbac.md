# Role Based Access Control

k0rdent provides the opportunity to use Role Based Access Control in order to try to use the principle of least privilege and only give users access to the objects and resources they absolutely have to have.

## What roles do
k0rdent leverages the Kubernetes RBAC system and provides a set of standard `ClusterRoles` with
associated permissions. These standard `ClusterRoles` are created as part of the k0rdent helm chart.
k0rdent roles are based on labels and aggregated permissions, meaning they automatically collect
rules from other `ClusterRoles` with specific labels.

The following table outlines the roles available in k0rdent, along with their respective read/write or read-only
permissions:

| Roles                            | Global Admin | Global Viewer | Namespace Admin | Namespace Editor | Namespace Viewer |
|----------------------------------|--------------|---------------|-----------------|------------------|------------------|
| **Scope**                        | **Global**   | **Global**    | **Namespace**   | **Namespace**    | **Namespace**    |
| k0rdent management               | r/w          | r/o           | -               | -                | -                |
| Namespaces management            | r/w          | r/o           | -               | -                | -                |
| Provider Templates               | r/w          | r/o           | -               | -                | -                |
| Global Template Management       | r/w          | r/o           | -               | -                | -                |
| Multi Cluster Service Management | r/w          | r/o           | -               | -                | -                |
| Template Chain Management        | r/w          | r/o           | r/w             | r/o              | r/o              |
| Cluster and Service Templates    | r/w          | r/o           | r/w             | r/o              | r/o              |
| Credentials                      | r/w          | r/o           | r/w             | r/o              | r/o              |
| Flux Helm objects                | r/w          | r/o           | r/w             | r/o              | r/o              |
| Cluster Deployments              | r/w          | r/o           | r/w             | r/w              | r/o              |


This section provides an overview of all `ClusterRoles` available in k0rdent.

## Roles summary

> NOTE:
> The names of the `ClusterRoles` may have different prefix depending on the name of the k0rdent Helm chart.
> The `ClusterRoles` definitions below use the `kcm` prefix, which is the default name of the k0rdent Helm chart.

### Global Admin

The `Global Admin` role provides full administrative access across all the k0rdent system.

**Name**: `kcm-global-admin-role`

**Aggregation Rule**: Includes all `ClusterRoles` with the labels:

* `k0rdent.mirantis.com/aggregate-to-global-admin: true`
* `k0rdent.mirantis.com/aggregate-to-namespace-admin: true`
* `k0rdent.mirantis.com/aggregate-to-namespace-editor: true`

**Permissions**:

1. Full access to k0rdent API
2. Full access to Flux Helm repositories and Helm charts
3. Full access to Cluster API identities
4. Full access to namespaces and secrets

**Use case**

A user with the `Global Admin` role is authorized to perform the following actions:

1. Manage the k0rdent configuration
2. Manage namespaces in the management cluster
3. Manage `Provider Templates`: add new templates or remove unneeded ones
4. Manage `Cluster` and `Service Templates` in any namespace, including adding and removing templates
5. Manage Flux `HelmRepositories` and `HelmCharts` in any namespace
6. Manage access rules for `Cluster` and `Service Templates`, including distributing templates across namespaces using
   `Template Chains`
7. Manage upgrade sequences for `Cluster` and `Service Templates`
8. Manage and deploy Services across multiple clusters in any namespace by modifying `MultiClusterService` resources
9. Manage `ClusterDeployments` in any namespace
10. Manage `Credentials` and `secrets` in any namespace
11. Upgrade k0rdent
12. Uninstall k0rdent


### Global Viewer

The `Global Viewer` role grants read-only access across the k0rdent system. It does not permit any modifications,
including the creation of clusters.

**Name**: `kcm-global-viewer-role`

**Aggregation Rule**: Includes all `ClusterRoles` with the labels:

* `k0rdent.mirantis.com/aggregate-to-global-viewer: true`
* `k0rdent.mirantis.com/aggregate-to-namespace-viewer: true`

**Permissions**:

1. Read access to k0rdent API
2. Read access to Flux Helm repositories and Helm charts
3. Read access to Cluster API identities
4. Read access to namespaces and secrets

**Use case**

A user with the `Global Viewer` role is authorized to perform the following actions:

1. View the k0rdent configuration
2. List namespaces available in the management cluster
3. List and get the detailed information about available `Provider Templates`
4. List available `Cluster` and `Service Templates` in any namespace
5. List and view detailed information about Flux `HelmRepositories` and `HelmCharts` in any namespace
6. View access rules for `Cluster` and `Service Templates`, including `Template Chains` in any namespace
7. View full details about the created `MultiClusterService` objects
8. List and view detailed information about `ClusterDeployments` in any namespace
9. List and view detailed information about created `Credentials` and `secrets` in any namespace


### Namespace Admin

The `Namespace Admin` role provides full administrative access within a namespace.

**Name**: `kcm-namespace-admin-role`

**Aggregation Rule**: Includes all `ClusterRoles` with the labels:

* `k0rdent.mirantis.com/aggregate-to-namespace-admin: true`
* `k0rdent.mirantis.com/aggregate-to-namespace-editor: true`

**Permissions**:

1. Full access to `ClusterDeployments`, `Credentials`, `Cluster` and `Service Templates` in the namespace
2. Full access to `Template Chains` in the namespace
3. Full access to Flux `HelmRepositories` and `HelmCharts` in the namespace

**Use case**

A user with the `Namespace Admin` role is authorized to perform the following actions within the namespace:

1. Create and manage all `ClusterDeployments` in the namespace
2. Create and manage `Cluster` and `Service Templates` in the namespace
3. Manage the distribution and upgrade sequences of Templates within the namespace
4. Create and manage Flux `HelmRepositories` and `HelmCharts` in the namespace
5. Manage `Credentials` created by any user in the namespace


### Namespace Editor

The `Namespace Editor` role allows users to create and modify `ClusterDeployments` within namespace using predefined
`Credentials` and `Templates`.

**Name**: `kcm-namespace-editor-role`

**Aggregation Rule**: Includes all `ClusterRoles` with the labels:

* `k0rdent.mirantis.com/aggregate-to-namespace-editor: true`

**Permissions**:

1. Full access to `ClusterDeployments` in the allowed namespace
2. Read access to `Credentials`, `Cluster` and `Service Templates`, and `TemplateChains` in the namespace
3. Read access to Flux `HelmRepositories` and `HelmCharts` in the namespace

**Use case**

A user with the `Namespace Editor` role has the following permissions in the namespace:

1. Can create and manage `ClusterDeployment` objects in the namespace using existing `Credentials` and `Templates`
2. Can list and view detailed information about the `Credentials` available in the namespace
3. Can list and view detailed information about the available `Cluster` and `Service Templates` and the `Templates'`
   upgrade sequences
4. Can list and view detailed information about the Flux `HelmRepositories` and `HelmCharts`


### Namespace Viewer

The `Namespace Viewer` role grants read-only access to resources within a namespace.

**Name**: `kcm-namespace-viewer-role`

**Aggregation Rule**: Includes all `ClusterRoles` with the labels:

* `k0rdent.mirantis.com/aggregate-to-namespace-viewer: true`

**Permissions**:

1. Read access to `ClusterDeployments` in the namespace
2. Read access to `Credentials`, `Cluster` and `Service Templates`, and `TemplateChains` in the namespace
3. Read access to Flux `HelmRepositories` and `HelmCharts` in the namespace

**Use case**

A user with the `Namespace Viewer` role has the following permissions in the namespace:

1. Can list and view detailed information about all the `ClusterDeployment` objects in the allowed namespace
2. Can list and view detailed information about `Credentials` available in the specific namespace
3. Can list and view detailed information about available `Cluster` and `Service Templates` and the `Templates'`
   upgrade sequences
4. Can list and view detailed information about Flux `HelmRepositories` and `HelmCharts`