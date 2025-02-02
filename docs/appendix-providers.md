# Cloud Provider Credentials Management in CAPI

Cloud provider credentials in Cluster API (CAPI) environments are managed through objects in the management cluster. Three objects handle credential storage and management, while a fourth object renders configuration into child clusters.

The configuration follows two patterns:
    - The first uses a ClusterIdentity resource that defines provider identity configuration and references a Secret with credentials. This approach is used by Azure and vSphere providers.
    - The second uses only a Source Secret without ClusterIdentity, as used by OpenStack, where the Secret contains all configurations.

The Credential resource provides an abstraction layer, referencing either a ClusterIdentity through identityRef or directly referencing a Secret, depending on the pattern used.

The Template ConfigMap, marked with projectsveltos.io/template: "true", contains Go template code generating child cluster resources. Template processing accesses infrastructure cluster objects through built-in Sveltos variables, and/or through the getResource function for additionally exposed objects.

The templating system uses Sprig functions for string manipulation, encoding/decoding, and type conversion. For details see Sprig docs at https://masterminds.github.io/sprig/ and Sveltos examples at https://projectsveltos.github.io/sveltos/template/intro_template/.
Additional template examples can be found in the repository for each supported CAPI provider at https://github.com/k0rdent/kcm/tree/main/config/dev, in `*.credentials.yaml` files.

## Provider Registration
New providers are registered through YAML configuration files mounted into the manager container at startup.

The provider configuration files must be mounted into a predefined path in the manager container, typically done through a ConfigMap. Provider configuration examples can be found at https://github.com/k0rdent/kcm/tree/main/providers.
