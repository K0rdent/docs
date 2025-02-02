# QuckStart 1 - Set up Management Node and Cluster

Please review the [Guide to QuickStarts](guide-to-quickstarts.md) for preliminaries. This QuickStart unit details setting up a single-VM environment for managing and interacting with k0rdent, and for hosting k0rdent components on a single-node local Kubernetes management cluster. Once k0rdent is installed on the management cluster, you can drive k0rdent by SSHing into the management node (kubectl is there and will be provisioned with the appropriate kubeconfig) or remotely by various means (e.g., install the management cluster kubeconfig in Lens or another Kubernetes dashboard on your laptop, tunnel across from your own local kubectl, etc.)

## Install a single-node k0s cluster locally to work as k0rdent's management cluster

[k0s Kubernetes](https://k0sproject.io) is a CNCF-certified minimal single-binary Kubernetes that installs with one command, and brings along its own CLI. We're using it to quickly set up a single-node management cluster on our manager node. However, k0rdent works on any CNCF-certified Kubernetes. If you choose to use something else, Team k0rdent would love to hear how you set things up to work for you.

```shell
curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo sh
sudo k0s install controller --single
sudo k0s start
```

You can check to see if the cluster is working by leveraging kubectl (installed and configured automatically by k0s) via the k0s CLI:

```shell
sudo k0s kubectl get nodes
```

You should see something like this:

```console
NAME              STATUS   ROLES    AGE   VERSION
ip-172-31-29-61   Ready    <none>   46s   v1.31.2+k0s
```

## Install kubectl

k0s installs a compatible kubectl and makes it accessible via its own client. But to make your environment easier to configure, we advise installing kubectl the normal way on the manager node and using it to control the local k0s management cluster.

```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly
sudo apt-get update
sudo apt-get install -y kubectl
```

## Get the local k0s cluster's kubeconfig for kubectl

On startup, k0s stores the administrator's kubeconfig in a local directory, making it easy to access:

```shell
sudo cp /var/lib/k0s/pki/admin.conf KUBECONFIG
sudo chmod +r KUBECONFIG
export KUBECONFIG=./KUBECONFIG
```

At this point, your newly-installed kubectl should be able to interoperate with the k0s management cluster with administrative privileges. Test to see that the cluster is ready (usually takes about one minute):

```shell
kubectl get nodes
```

You should see something like this:

```console
NAME              STATUS   ROLES           AGE   VERSION
ip-172-31-29-61   Ready    control-plane   25m   v1.31.2+k0s
```

## Install Helm

The Helm Kubernetes package manager is used to install k0rdent services. We'll install Helm as follows:

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Issuing these commands should produce something very much like the following output:

```console
Downloading https://get.helm.sh/helm-v3.16.3-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm
```

## Install k0rdent into the k0s management cluster

Now we'll install k0rdent itself into the k0s management cluster:

```shell
helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm --version 0.1.0 -n kcm-system --create-namespace
```

You'll see something like the following. Ignore the warnings, since this is an ephemeral, non-production, non-shared environment:

```console
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: ./KUBECONFIG
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: ./KUBECONFIG
Pulled: ghcr.io/k0rdent/kcm/charts/kcm:0.1.0
Digest: sha256:1f75e8e55c44d10381d7b539454c63b751f9a2ec6c663e2ab118d34c5a21087f
NAME: kcm
LAST DEPLOYED: Mon Dec  9 00:32:14 2024
NAMESPACE: kcm-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

k0rdent startup takes several minutes.

## Check that k0rdent cluster management pods are running

One fundamental k0rdent subsystem, k0rdent Cluster Manager (KCM), handles cluster lifecycle management on clouds and infrastructures: i.e., it helps you configure and compose clusters and manages infrastructure via Cluster API (CAPI). Before continuing, check that KCM pods are ready:

```shell
kubectl get pods -n kcm-system   # check pods in the kcm-system namespace
```

You should see something like:

```console
NAME                                                           READY   STATUS
azureserviceoperator-controller-manager-86d566cdbc-rqkt9       1/1     Running
capa-controller-manager-7cd699df45-28hth                       1/1     Running
capi-controller-manager-6bc5fc5f88-hd8pv                       1/1     Running
capv-controller-manager-bb5ff9bd5-7dsr9                        1/1     Running
capz-controller-manager-5dd988768-qjdbl                        1/1     Running
helm-controller-76f675f6b7-4d47l                               1/1     Running
kcm-cert-manager-7c8bd964b4-nhxnq                              1/1     Running
kcm-cert-manager-cainjector-56476c46f9-xvqhh                   1/1     Running
kcm-cert-manager-webhook-69d7fccf68-s46w8                      1/1     Running
kcm-cluster-api-operator-79459d8575-2s9jc                      1/1     Running
kcm-controller-manager-64869d9f9d-zktgw                        1/1     Running
k0smotron-controller-manager-bootstrap-6c5f6c7884-d2fqs        2/2     Running
k0smotron-controller-manager-control-plane-857b8bffd4-zxkx2    2/2     Running
k0smotron-controller-manager-infrastructure-7f77f55675-tv8vb   2/2     Running
source-controller-5f648d6f5d-7mhz5                             1/1     Running
```

Pods reported in states other than Running should become ready momentarily.

## Check that the projectsveltos pods are running

The other fundamental k0rdent subsystem, k0rdent Service Manager (KSM), handles services configuration and lifecycle management on clusters. This utilizes the [projectsveltos](https://github.com/projectsveltos) Kubernetes Add-On Controller and other open source projects. Before continuing, check that KSM pods are ready:

```shell
kubectl get pods -n projectsveltos   # check pods in the projectsveltos namespace
```

You should see something like:

```console
NAME                                     READY   STATUS    RESTARTS   AGE
access-manager-cd49cffc9-c4q97           1/1     Running   0          16m
addon-controller-64c7f69796-whw25        1/1     Running   0          16m
classifier-manager-574c9d794d-j8852      1/1     Running   0          16m
conversion-webhook-5d78b6c648-p6pxd      1/1     Running   0          16m
event-manager-6df545b4d7-mbjh5           1/1     Running   0          16m
hc-manager-7b749c57d-5phkb               1/1     Running   0          16m
sc-manager-f5797c4f8-ptmvh               1/1     Running   0          16m
shard-controller-767975966-v5qqn         1/1     Running   0          16m
sveltos-agent-manager-56bbf5fb94-9lskd   1/1     Running   0          15m
```

If you have fewer pods than shown above, just wait a little while for all the pods to reconcile and start running.

## Verify that KCM provider and related templates are available

k0rdent KCM leverages CAPI to manage Kubernetes cluster assembly and host infrastructure. CAPI requires infrastructure providers for different clouds and infrastructure types. These are delivered and referenced within k0rdent using templates, instantiated in the management cluster as objects. Before continuing, verify that default provider template objects are installed and verified. Other templates are also stored as provider templates in this namespace &mdash; for example, the templates that determine setup of KCM itself and other parts of the k0rdent system (e.g., projectsveltos, which is a component of k0rdent Service Manager (KSM, see below)) as well as the k0smotron subsystem, which enables creation and lifecycle management of managed clusters that use Kubernetes-hosted control planes (i.e., control planes as pods):

```shell
kubectl get providertemplate -n kcm-system   # list providertemplate objects in the kcm-system namespace
```

You should see output similar to what's shown below. The placeholder X-Y-Z represents the current version number of the template, and will be replaced in the listing with digits:

```console
NAME                                 VALID
cluster-api-X-Y-Z                    true
cluster-api-provider-aws-X-Y-Z       true
cluster-api-provider-azure-X-Y-Z     true
cluster-api-provider-vsphere-X-Y-Z   true
kcm-X-Y-Z                            true
k0smotron-X-Y-Z                      true
projectsveltos-X-Y-Z                 true
```

## Verify that KCM ClusterTemplate objects are available

CAPI also requires control plane and bootstrap (worker node) providers to construct and/or manage different Kubernetes cluster distros and variants. Again, these providers are delivered and referenced within k0rdent using templates, instantiated in the management cluster as `ClusterTemplate` objects. Before continuing, verify that default ClusterTemplate objects are installed and verified:

```shell
kubectl get clustertemplate -n kcm-system   # list clustertemplate objects in the kcm-system namespace
```

You should see output similar to what's shown below:

```console
NAME                                VALID
aws-eks-X-Y-Z                       true
aws-hosted-cp-X-Y-Z                 true
aws-standalone-cp-X-Y-Z             true
azure-hosted-cp-X-Y-Z               true
azure-standalone-cp-X-Y-Z           true
vsphere-hosted-cp-X-Y-Z             true
vsphere-standalone-cp-X-Y-Z         true
```

## Verify that KSM ServiceTemplate objects are available

k0rdent Service Manager (KSM) uses Service Templates to lifecycle manage services and applications installed on clusters. These, too, are represented as declarative templates, instantiated as ServiceTemplate objects. Check that default ServiceTemplate objects have been created and validated:

```shell
kubectl get servicetemplate -n kcm-system   # list servicetemplate objects in the kcm-system namespace
```

You should see output similar to what's shown below:

```console
NAME                      VALID
cert-manager-1-16-2       true
dex-0-19-1                true
external-secrets-0-11-0   true
ingress-nginx-4-11-0      true
ingress-nginx-4-11-3      true
kyverno-3-2-6             true
velero-8-1-0              true
```
## Next steps

Your QuickStart management node is now complete, and k0rdent is installed and operational. Next, it's time to select [AWS](quickstart-2-aws.md) or [Azure](quickstart-2-azure.md) as an environment for hosting managed clusters.
