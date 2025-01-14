# Upgrading k0rdent to the newer version

> NOTE: To upgrade k0rdent the user must have `Global Admin` role.
> For the detailed information about k0rdent RBAC, refer to the [RBAC documentation](../rbac/roles.md).

Follow the steps below to update k0rdent to a newer version:

**Step 1. Create a New `Release` Object**

Create a `Release` object in the management cluster for the desired version. For example, to create
a `Release` for version `v0.0.7`, run the following command:

```shell
VERSION=v0.0.7
kubectl create -f https://github.com/k0rdent/kcm/releases/download/${VERSION}/release.yaml
```

**Step 2. Update the `Management` Object with the New `Release`**

- List available `Releases`:

To view all available `Releases`, run:

```shell
kubectl get releases
```

Example output:

```shell
NAME        AGE
kcm-0-0-6   71m
kcm-0-0-7   65m
```

- Patch the `Management` Object with the New `Release` Name:

Update the `spec.release` field in the `Management` object to point to the new release. Replace `kcm-0-0-4` with
the name of your new release:

```shell
RELEASE_NAME=kcm-0-0-7
kubectl patch management.kcm kcm --patch "{\"spec\":{\"release\":\"${RELEASE_NAME}\"}}" --type=merge
```

**Step 3. Verify the Upgrade**

Check the status of the `Management` object to monitor the readiness of the components:

```shell
kubectl get management.kcm kcm -o=jsonpath={.status} | jq
```
