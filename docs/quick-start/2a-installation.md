
### Requirements

2A requires a k8s cluster, it can be of any type, it will become the 2A management cluster.

If you don't have a k8s cluster yet, we suggest to use [k0s](https://docs.k0sproject.io/stable/install/).

The below commands assume:

- That your kubeconfig points to the correct Kubernetes Cluster.
- You have [helm](https://helm.sh/docs/intro/install/) installed.
- You have [kubectl](https://kubernetes.io/docs/tasks/tools/) installed.

### Installation via helm

```bash
helm install hmc oci://ghcr.io/mirantis/hmc/charts/hmc --version 0.0.3 -n hmc-system --create-namespace
```

### Verification

The installation will take a couple of minutes until 2A and its subcomponents is fully installed and configured.

You can verify the successfull installation with checking all the pods in the `hmc-system` namespace, the output should rougly look like:

```bash
kubectl -n hmc-system get pods -n hmc-system

NAME                                                           READY   STATUS
azureserviceoperator-controller-manager-86d566cdbc-rqkt9       1/1     Running
capa-controller-manager-7cd699df45-28hth                       1/1     Running
capi-controller-manager-6bc5fc5f88-hd8pv                       1/1     Running
capv-controller-manager-bb5ff9bd5-7dsr9                        1/1     Running
capz-controller-manager-5dd988768-qjdbl                        1/1     Running
helm-controller-76f675f6b7-4d47l                               1/1     Running
hmc-cert-manager-7c8bd964b4-nhxnq                              1/1     Running
hmc-cert-manager-cainjector-56476c46f9-xvqhh                   1/1     Running
hmc-cert-manager-webhook-69d7fccf68-s46w8                      1/1     Running
hmc-cluster-api-operator-79459d8575-2s9jc                      1/1     Running
hmc-controller-manager-64869d9f9d-zktgw                        1/1     Running
k0smotron-controller-manager-bootstrap-6c5f6c7884-d2fqs        2/2     Running
k0smotron-controller-manager-control-plane-857b8bffd4-zxkx2    2/2     Running
k0smotron-controller-manager-infrastructure-7f77f55675-tv8vb   2/2     Running
source-controller-5f648d6f5d-7mhz5                             1/1     Running
```

If you have less pods, give 2A a little longer to reconcile all the pods.

As a second verification, check that the example ClusterTemplates have been installed and are valid:

```bash

kubectl get clustertemplate -n hmc-system

NAME                                VALID
aws-eks-0-0-1                       true
aws-hosted-cp-0-0-2                 true
aws-standalone-cp-0-0-1             true
aws-standalone-cp-0-1-0             true
azure-hosted-cp-0-0-3               true
azure-standalone-cp-0-0-1           true
azure-standalone-cp-0-1-0           true
remote-single-standalone-cp-0-1-4   true
remote-single-standalone-cp-0-1-5   true
remote-single-standalone-cp-0-1-6   true
remote-single-standalone-cp-0-1-7   true
remote-single-standalone-cp-0-1-8   true
vsphere-hosted-cp-0-0-2             true
vsphere-standalone-cp-0-0-2         true
```

### Next Step

Now you can start to configure our Infra Provider of choice and create the first Managed Cluster.