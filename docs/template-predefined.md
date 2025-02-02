## Remove Templates shipped with k0rdent

If you need to limit the templates that exist in your k0rdent installation, follow the instructions below:

1. Get the list of `ProviderTemplates`, `ClusterTemplates` or `ServiceTemplates` shipped with k0rdent. For example,
for `ClusterTemplate` objects, run:

    ```bash
    kubectl get clustertemplates -n kcm-system -l helm.toolkit.fluxcd.io/name=kcm-templates
    ```
    ```console
    NAME                       VALID
    aws-hosted-cp              true
    aws-standalone-cp          true
    ```

2. Remove the template from the list using `kubectl delete`. For example:

    ```bash
    kubectl delete clustertemplate -n kcm-system <template-name>
    ```