# Upgrading k0rdent

Upgrading k0rdent involves making upgrades to the `Management` object. To do that, you must have the `Global Admin` role. (For detailed information about k0rdent RBAC roles and permissions, refer to the [RBAC documentation](admin-rbac.md).) Follow these steps to upgrade k0rdent:

1. Create a new `Release` object

    Start by creating a `Release` object in the management cluster the points to the desired version. You can see
    available versions at [https://github.com/k0rdent/kcm/releases](https://github.com/k0rdent/kcm/releases).  The actual
    `Release` object includes information on the templates and resources that are available, as well as the version of the
    Kubernetes Cluster API.  For example, the v0.0.7 `Release` object looks like this:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: Release
    metadata:
      name: kcm-0-0-7
    annotations:
      helm.sh/resource-policy: keep
    labels:
      k0rdent.mirantis.com/component: kcm
    spec:
      version: 0.0.7
      kcm:
        template: kcm-0-0-7
      capi:
        template: cluster-api-0-0-6
      providers:
        - name: k0smotron
          template: k0smotron-0-0-6
        - name: cluster-api-provider-azure
          template: cluster-api-provider-azure-0-0-4
        - name: cluster-api-provider-vsphere
          template: cluster-api-provider-vsphere-0-0-5
        - name: cluster-api-provider-aws
          template: cluster-api-provider-aws-0-0-4
        - name: cluster-api-provider-openstack
          template: cluster-api-provider-openstack-0-0-1
        - name: projectsveltos
          template: projectsveltos-0-45-0
    ```

    Thankfully, you don't have to build these YAML files yourself. Once you've chosen a release, you can go ahead and create the release object by referencing the YAML file online, as in:

    ```shell
    VERSION=v0.0.7
    kubectl create -f https://github.com/k0rdent/kcm/releases/download/${VERSION}/release.yaml
    ```

2. List Available `Releases`

    Once you've created the new `Release` you need to update the `Management` object to use it. Start by viewing all available `Release`s:

    ```shell
    kubectl get releases
    ```

    ```console
    NAME        AGE
    kcm-0-0-6   71m
    kcm-0-0-7   65m
    ```

3. Patch the `Management` object with the new `Release`

    Update the `spec.release` field in the `Management` object to point to the new release. Replace `<release-name>` with the name of your desired release:

    ```shell
    RELEASE_NAME=kcm-0-0-7
    kubectl patch management.kcm kcm --patch "{\"spec\":{\"release\":\"${RELEASE_NAME}\"}}" --type=merge
    ```

4. Verify the Upgrade

    Although the change will be made immediately, it will take some time for k0rdent to update the components it should be
    using. Monitor the readiness of the `Management` object to ensure the upgrade was successful. For example:

    ```shell
    kubectl get management.kcm kcm
    NAME   READY   RELEASE     AGE
    kcm    True    kcm-0-0-7   4m34s
    ```
