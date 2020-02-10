# OpenShift 4.x Cloud Reference Architecture

# WORK IN PROGRESS 

## Installation Flow

Bootstrapping a cluster involves the following steps:

1. The bootstrap machine boots and starts hosting the remote resources required for the master machines to boot.
2. The master machines fetch the remote resources from the bootstrap machine and finish booting.
3. The master machines use the bootstrap machine to form an etcd cluster.
4. The bootstrap machine starts a temporary Kubernetes control plane using the new etcd cluster.
5. The temporary control plane schedules the production control plane to the master machines.
6. The temporary control plane shuts down and passes control to the production control plane.
7. The bootstrap machine injects OpenShift Container Platform components into the production control plane.
8. The installation program shuts down the bootstrap machine.
9. The control plane sets up the worker nodes.
10. The control plane installs additional services in the form of a set of Operators.

The result of this bootstrapping process is a fully running OpenShift Container Platform cluster. The cluster then downloads and configures remaining components needed for the day-to-day operation, including the creation of worker machines in supported environments.

### Disconnected Installation

You can mirror the contents of the OpenShift Container Platform registry and the images that are required to generate the installation program.

The mirror registry is a key component that is required to complete an installation in a restricted network. You can create this mirror on a bastion host, which can access both the internet and your closed network, or by using other methods that meet your restrictions.

Because of the way that OpenShift Container Platform verifies integrity for the release payload, the image references in your local registry are identical to the ones that are hosted by Red Hat on [Quay.io](https://quay.io/). During the bootstrapping process of installation, the images must have the same digests no matter which repository they are pulled from. To ensure that the release payload is identical, you mirror the images to your local repository.

RedHat provides [step by step instructions](https://docs.openshift.com/container-platform/4.2/installing/install_config/installing-restricted-networks-preparations.html) on creating an installation environment for installation in a restricted network.  The process can be summarized as:

1. Create a mirror registry.  You can build your own [using podman and a generic web server](https://docs.openshift.com/container-platform/4.2/installing/install_config/installing-restricted-networks-preparations.html#installation-creating-mirror-registry_installing-restricted-networks-preparations) or you can use a packaged image regustry solution like [Harbor](https://getharbor.io) or [Quay](https://quay.io) which provides image scanning and other advanced functionality.

2. Once your image registry is created, you need to [update your pull-secret file](https://docs.openshift.com/container-platform/4.2/installing/install_config/installing-restricted-networks-preparations.html#installation-adding-registry-pull-secret_installing-restricted-networks-preparations) with its authentication details.

3. Clone the official OpenShift 4.x [quay.io registry](https://quay.io/repository/openshift-release-dev/ocp-release?tab=tags.  The `oc` command line tool provides functionality that will pull all required artifacts from this external repository.

```shell
$ export OCP_RELEASE=<release_version>
$ export LOCAL_REGISTRY='<local_registry_host_name>:<local_registry_host_port>'
$ export LOCAL_REPOSITORY='<repository_name>'
$ export PRODUCT_REPO='openshift-release-dev'
$ export LOCAL_SECRET_JSON='<path_to_pull_secret>'
$ export RELEASE_NAME="ocp-release"

$ oc adm -a ${LOCAL_SECRET_JSON} release mirror \
     --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE} \
     --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
     --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}
```

If your registry is not protected with TLS, add the `--insecure` flag to the oc command.  Once this command completes, it will output several snippets that you must add to the installation configuration, and to your OpenShift cluster to provide updates.

```shell
$ export OCP_RELEASE="4.3.0-x86_64"
$ export LOCAL_REGISTRY="ncm-mbpr.local:80"
$ export LOCAL_REPOSITORY="openshift4/ocp4"
$ export PRODUCT_REPO='openshift-release-dev'
$ export LOCAL_SECRET_JSON='./pull-secret'
$ export RELEASE_NAME="ocp-release"

$ oc adm -a ${LOCAL_SECRET_JSON} release mirror --insecure \
     --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE} \
     --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
     --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}

...

info: Mirroring completed in 8m46.5s (10.38MB/s)

Success
Update image:  ncm-mbpr.local:80/openshift4/ocp4:4.3.0-x86_64
Mirror prefix: ncm-mbpr.local:80/openshift4/ocp4
```

To use the new mirrored repository to install, add the following section to the bottom of your install-config.yaml:

```
imageContentSources:

- mirrors:
  - ncm-mbpr.local:80/openshift4/ocp4
    source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - ncm-mbpr.local:80/openshift4/ocp4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

```
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - ncm-mbpr.local:80/openshift4/ocp4
      source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - ncm-mbpr.local:80/openshift4/ocp4
      source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```
