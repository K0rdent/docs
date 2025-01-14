# OpenStack Quick Start

Much of the following includes the process of setting up credentials for OpenStack.
To better understand how k0rdent uses credentials, read the
[Credential system](../credential/main.md).

## Prerequisites

### k0rdent Management Cluster

You need a Kubernetes cluster with [kcm installed](installation.md).

### Software prerequisites

- OpenStack CLI (optional)
- Application Credential in OpenStack (recommended for enhanced security)
    If you prefer username/password, adjust accordingly in your YAML.

## Step 1: Configure OpenStack Application Credential

This credential should include:

- OS_AUTH_URL
- OS_APPLICATION_CREDENTIAL_ID
- OS_APPLICATION_CREDENTIAL_SECRET
- OS_REGION_NAME
- OS_INTERFACE
- OS_IDENTITY_API_VERSION (commonly 3)
- OS_AUTH_TYPE (e.g., v3applicationcredential)

> NOTE:
> Using an Application Credential is strongly recommended because it limits scope and improves security over a raw username/password approach.

## Step 2: Create the OpenStack Credentials Secret on k0rdent Management Cluster

Create a Kubernetes Secret containing the clouds.yaml that defines your OpenStack environment. Save this as `openstack-cloud-config.yaml` (for example):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openstack-cloud-config
  namespace: kcm-system
stringData:
  clouds.yaml: |
    clouds:
      openstack:
        auth:
          auth_url: <OS_AUTH_URL>
          application_credential_id: <OS_APPLICATION_CREDENTIAL_ID>
          application_credential_secret: <OS_APPLICATION_CREDENTIAL_SECRET>
        region_name: <OS_REGION_NAME>
        interface: <OS_INTERFACE>
        identity_api_version: <OS_IDENTITY_API_VERSION>
        auth_type: <OS_AUTH_TYPE>
```

Apply the YAML to your cluster using the following command:

```bash
kubectl apply -f openstack-cloud-config.yaml
```

## Step 3: Create the k0rdent Credential Object

Next, define a Credential that references the Secret from Step 2.
Save this as `openstack-cluster-identity-cred.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Credential
metadata:
  name: openstack-cluster-identity-cred
  namespace: kcm-system
spec:
  description: "OpenStack credentials"
  identityRef:
    apiVersion: v1
    kind: Secret
    name: openstack-cloud-config
    namespace: kcm-system
```

Apply the YAML to your cluster:

```bash
kubectl apply -f openstack-cluster-identity-cred.yaml
```

> NOTE:
> 1. .spec.identityRef.kind hould be Secret.
> 2. .spec.identityRef.name must match the Secret you created in Step 2.
> 3. .spec.identityRef.namespace must be the same as the Secret’s namespace (kcm-system).

## Step 4: Create Your First Managed Cluster

Create a YAML with the specification of your Managed Cluster and save it as
`my-openstack-cluster-deployment.yaml`.

Here is an example:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-openstack-cluster-deployment
  namespace: kcm-system
spec:
  template: openstack-standalone-cp-0-0-1
  credential: openstack-cluster-identity-cred
  config:
    controlPlaneNumber: 1
    workersNumber: 1
    controlPlane:
      flavor: m1.medium
      image:
        filter:
          name: ubuntu-22.04-x86_64
    worker:
      flavor: m1.medium
      image:
        filter:
          name: ubuntu-22.04-x86_64
    authURL: <OS_AUTH_URL>
```

> NOTE:
> 1. spec.template references the OpenStack-specific blueprint (e.g., openstack-standalone-cp-0-0-1).
> 2. Adjust flavor, image name, and authURL to match your OpenStack environment.
> 3. For more information about the config options, see the [OpenStack Template Parameters](../clustertemplates/openstack/template-parameters.md).

Apply the YAML to your management cluster:

```bash
kubectl apply -f my-openstack-cluster-deployment.yaml
```

There will be a delay as the cluster finishes provisioning. Follow the
provisioning process with the following command:

```bash
kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com my-openstack-cluster-deployment --watch
```

After the cluster is `Ready`, you can access it via the kubeconfig, like this:

```bash
kubectl -n kcm-system get secret my-openstack-cluster-deployment-kubeconfig -o jsonpath='{.data.value>' | base64 -d > my-openstack-cluster-deployment-kubeconfig.kubeconfig
```

```bash
KUBECONFIG="my-openstack-cluster-deployment-kubeconfig.kubeconfig" kubectl get pods -A
```
