# Pre-defined ServiceTemplates

kcm comes with a set of pre-defined ServiceTemplates. These templates are referring to the following Helm charts:

> NOTE:
> Listed Helm charts will be looked up in the registry configured by the `--default-registry-url`
> flag provided to the `kcm-controller-manager` (default value: `oci://ghcr.io/k0rdent/kcm/charts`).
> In case of using different registry, the corresponding Helm charts must be available
> in the registry in prior to using the ServiceTemplate.

| Application                                             | Chart version | Source                                                                                     |
|---------------------------------------------------------|---------------|--------------------------------------------------------------------------------------------|
| [cert-manager](https://cert-manager.io/)                | 1.16.2        | [https://charts.jetstack.io](https://charts.jetstack.io)                                   |
| [external-secrets](https://external-secrets.io/latest/) | 0.11.0        | [https://charts.external-secrets.io](https://charts.external-secrets.io)                   |
| [dex](https://dexidp.io/)                               | 0.19.1        | [https://charts.dexidp.io](https://charts.dexidp.io)                                       |
| [velero](https://velero.io/)                            | 8.1.0         | [https://vmware-tanzu.github.io/helm-charts/](https://vmware-tanzu.github.io/helm-charts/) |
| [ingress-nginx](https://kubernetes.io/)                 | 4.11.0        | [https://kubernetes.github.io/ingress-nginx](https://kubernetes.github.io/ingress-nginx)   |
| [ingress-nginx](https://kubernetes.io/)                 | 4.11.3        | [https://kubernetes.github.io/ingress-nginx](https://kubernetes.github.io/ingress-nginx)   |
| [kyverno](https://kyverno.io/)                          | 3.2.6         | [https://kyverno.github.io/kyverno/](https://kyverno.github.io/kyverno/)                   |

## Configuration Requirements

The pre-defined ServiceTemplates are provided without any pre-configured values. Several of the listed charts require additional configuration before they can be effectively used:

- **dex**: Requires issuer URL, storage type and at least one connector to be configured
- **velero**: Requires provider's credentials to be configured
- **cert-manager**: Requires CRDs to be installed

Before using these ServiceTemplates, make sure to review each chart's documentation and configure the necessary values according to your environment requirements.


