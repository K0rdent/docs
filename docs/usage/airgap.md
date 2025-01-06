# Air-gapped Installation Guide

## Prerequisites

In order to install HMC in an air-gapped environment, you need will need the
following:

- An installed k0s cluster that will be used as the management cluster.  If you
  do not yet have a k0s cluster, you can follow the [Airgapped Installation](https://docs.k0sproject.io/head/airgap-install/#airgap-install)
  documentation.  k0s is recommended for airgapped installations because it
  implements an OCI image bundle watcher which allows k0s to utilize a bundle
  of management cluster images easily. Any Kubernetes distribution can be
  used, but instructions for using k0s are provided here.
- The `KUBECONFIG` of a management cluster that will be the target for the HMC
  installation.
- A registry that is accessible from the airgapped hosts to store the HMC images.
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

- A registry and associated chart repository for hosting HMC charts.  At this
  time all HMC charts MUST be hosted in a single OCI chart repository.  See
  [Use OCI-based registries](https://helm.sh/docs/topics/registries/) in the
  Helm documentation for more information.
- [jq](https://jqlang.github.io/jq/download/), Helm and Docker binaries
  installed on the machine where the `airgap-push.sh` script will be run.


## Installation

1. Download the HMC airgap bundle, the bundle contains the
following:

    - `images/hmc-images-<version>.tgz` - The image bundle tarball for the
      management cluster, this bundle will be loaded into the management
      cluster.
    - `images/hmc-extension-images-<version>.tgz` - The image bundle tarball for
      the managed clusters, this bundle will be pushed to a registry where the
      images can be accessed by the managed clusters.
    - `charts` - Contains the HMC Helm chart, dependency charts and k0s
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
     tar xvf hmc-airgap-<version>.tgz scripts/airgap-push.sh
     ./scripts/airgap-push.sh -r <registry> -c <chart-repo> -a hmc-airgap-<version>.tgz
     ```

3. Next, extract the `management` bundle tarball and sync the images to the
   k0s cluster which will host the management cluster.  See [Sync the Bundle File](https://docs.k0sproject.io/head/airgap-install/#2a-sync-the-bundle-file-with-the-airgapped-machine-locally)
   for more information.

     > NOTE:
     > Multiple image bundles can be placed in the `/var/lib/k0s/images`
     > directory for k0s to use and the existing `k0s` airgap bundle does not
     > need to be merged into the `hmc-images-<version>.tgz` bundle.

     ```bash
     tar -C /var/lib/k0s -xvf hmc-airgap-<version>.tgz "images/hmc-images-<version>.tgz"
     ```

4. Install the HMC Helm chart on the management cluster from the registry where
   the HMC charts were pushed.  The HMC controller image is loaded as part of
   the airgap `management` bundle and does not need to be customized within the
   Helm chart, but the default chart repository configured via
   `controller.defaultRegistryURL` should be set to reference the repository
   where charts have been pushed.

      ```bash
      helm install hmc oci://<chart-repository>/hmc \
        --version <hmc-version> \
        -n hmc-system \
        --create-namespace \
        --set controller.defaultRegistryURL="oci://<chart-repository>"
      ```

5. Within the `spec:` for your desired `ClusterDeployment` object, specify the
   custom image registry and chart repository to be used (the registry and chart
   repository where the `extensions` bundle and charts were pushed).

     ```yaml
     spec:
      config:
        extensions:
         imageRepository: ${IMAGE_REPOSITORY}
         chartRepository: ${CHART_REPOSITORY}
     ```
