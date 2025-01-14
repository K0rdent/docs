# Air-gapped Installation Guide

> WARNING:
> Currently only vSphere infrastructure provider supports full air-gapped
> installation.

## Prerequisites

In order to install kcm in an air-gapped environment, you need will need the
following:

- An installed k0s cluster that will be used as the management cluster.  If you
  do not yet have a k0s cluster, you can follow the [Airgapped Installation](https://docs.k0sproject.io/head/airgap-install/#airgap-install)
  documentation.  k0s is recommended for airgapped installations because it
  implements an OCI image bundle watcher which allows k0s to utilize a bundle
  of management cluster images easily. Any Kubernetes distribution can be
  used, but instructions for using k0s are provided here.
- The `KUBECONFIG` of a management cluster that will be the target for the kcm
  installation.
- A registry that is accessible from the airgapped hosts to store the kcm images.
  If you do not have a registry you can deploy a [local Docker registry](https://distribution.github.io/distribution/)
  or use [mindthegap](https://github.com/mesosphere/mindthegap?tab=readme-ov-file#serving-a-bundle-supports-both-image-or-helm-chart)

    > WARNING:
    > If using a local Docker registry, ensure the registry URL is added to
    > the `insecure-registries` key within the Docker `/etc/docker/daemon.json`
    > file.
    > ```json
    > {
    >   "insecure-registries": ["<registry-url>"]
    > }
    > ```

- A registry and associated chart repository for hosting kcm charts.  At this
  time all kcm charts MUST be hosted in a single OCI chart repository.  See
  [Use OCI-based registries](https://helm.sh/docs/topics/registries/) in the
  Helm documentation for more information.
- [jq](https://jqlang.github.io/jq/download/), Helm and Docker binaries
  installed on the machine where the `airgap-push.sh` script will be run.


## Installation

1. Download the kcm airgap bundle, the bundle contains the
following:

    - `images/kcm-images-<version>.tgz` - The image bundle tarball for the
      management cluster, this bundle will be loaded into the management
      cluster.
    - `images/kcm-extension-images-<version>.tgz` - The image bundle tarball for
      the managed clusters, this bundle will be pushed to a registry where the
      images can be accessed by the managed clusters.
    - `charts` - Contains the kcm Helm chart, dependency charts and k0s
      extensions charts within the `extensions` directory.  All of these charts
      will be pushed to a chart repository within a registry.
    - `scripts/airgap-push.sh` - A script that will aid in re-tagging and
      pushing the `ClusterDeployment` required charts and images to a desired
      registry.

2. Extract and use the `airgap-push.sh` script to push the `extensions` images
   and `charts` contents to the registry.  Ensure you have logged into the
   registry using both `docker login` and `helm registry login` before running
   the script.

     ```bash
     tar xvf kcm-airgap-<version>.tgz scripts/airgap-push.sh
     ./scripts/airgap-push.sh -r <registry> -c <chart-repo> -a kcm-airgap-<version>.tgz
     ```

3. Next, extract the `management` bundle tarball and sync the images to the
   k0s cluster which will host the management cluster.  See [Sync the Bundle File](https://docs.k0sproject.io/head/airgap-install/#2a-sync-the-bundle-file-with-the-airgapped-machine-locally)
   for more information.

     > NOTE:
     > Multiple image bundles can be placed in the `/var/lib/k0s/images`
     > directory for k0s to use and the existing `k0s` airgap bundle does not
     > need to be merged into the `kcm-images-<version>.tgz` bundle.

     ```bash
     tar -C /var/lib/k0s -xvf kcm-airgap-<version>.tgz "images/kcm-images-<version>.tgz"
     ```

4. Install the kcm Helm chart on the management cluster from the registry where
   the kcm charts were pushed.  The kcm controller image is loaded as part of
   the airgap `management` bundle and does not need to be customized within the
   Helm chart, but the default chart repository configured via
   `controller.defaultRegistryURL` should be set to reference the repository
   where charts have been pushed.

      ```bash
      helm install kcm oci://<chart-repository>/kcm \
        --version <version> \
        -n kcm-system \
        --create-namespace \
        --set controller.defaultRegistryURL=oci://<chart-repository>
      ```

5. Edit the `Management` object to add the airgap parameters.

	 > NOTE:
	 > Use `insecureRegistry` parameter only in case if you have plain HTTP
	 > registry.

     The resulting yaml may look like this:

      ```yaml
      apiVersion: k0rdent.mirantis.com/v1alpha1
      kind: Management
      metadata:
        name: kcm
      spec:
        core:
          capi:
            config:
              airgap: true
          kcm:
            config:
              controller:
                defaultRegistryURL: oci://<registry-url>
                insecureRegistry: true
        providers:
        - config:
            airgap: true
          name: k0smotron
        - config:
            airgap: true
          name: cluster-api-provider-vsphere
        - name: projectsveltos
        release: <release name>
      ```

6. Place k0s binary and airgap bundle at internal server, so they could be
   available over HTTP. This is required for the airgap provisioning process,
   since k0s components must be downloaded at each node upon creation.
   Alternatively you can create the following example deployment using the k0s
   image provided in the bundle.

      > NOTE:
      > k0s image version is the same that the default defined in the vSphere
      > template.


      ```yaml
	  ---
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: k0s-ag-image
        labels:
          app: k0s-ag-image
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: k0s-ag-image
        template:
          metadata:
            labels:
              app: k0s-ag-image
          spec:
            containers:
            - name: k0s-ag-image
              image: k0s-ag-image:v1.31.1-k0s.1
              ports:
              - containerPort: 80
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: k0s-ag-image
      spec:
        ports:
        - name: http
          port: 80
          protocol: TCP
          targetPort: 80
        selector:
          app: k0s-ag-image
        type: NodePort
	  ```

## Creation of the ClusterDeployment

In order to successfully deploy a cluster several configuration options must be
defined in the `.spec.config` of the `ClusterDeployment.

You must specify the custom image registry and chart repository to be used (the
registry and chart repository where the `extensions` bundle and charts were
pushed).

Apart from that you must provide endpoint where k0s binary and airgap bundle
could be downloaded (step `6` of the [installation procedure](#installation))

```yaml
spec:
 config:
   airgap: true
   k0s:
     downloadURL: "http://<k0s binary endpoint>/k0s"
     bundleURL: "http://<k0s binary endpoint>/k0s-airgap-bundle"
   extensions:
    imageRepository: ${IMAGE_REPOSITORY}
    chartRepository: ${CHART_REPOSITORY}
```
