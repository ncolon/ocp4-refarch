# OpenShift 4.x Cloud Reference Architecture

* [Resource Management](#resource-management)
* [Network](#network)
   * [Subnets](#subnets)
   * [Availability Zones](#availability-zones)
   * [LoadBalancers](#loadbalancers)
   * [Security Groups](#security-groups)
* [DNS](#dns)
   * [Private DNS Zone](#private-dns-zone)
   * [Public DNS Zone](#public-dns-zone)
* [Access Management](#access-management)
* [Cluster Nodes](#cluster-nodes)
   * [Control Plane Nodes (masters)](#control-plane-nodes-masters)
   * [Compute Nodes (workers)](#compute-nodes-workers)
   * [Recommended Resource Sizing](#recommended-resource-sizing)
   * [OS Image](#os-image)
* [Storage](#storage)
   * [Compute Node Storage](#compute-node-storage)
   * [Persistent Volume for Image Registry](#persistent-volume-for-image-registry)
   * [Persistent Volumes for Workloads](#persistent-volumes-for-workloads)
      * [Access Modes](#access-modes)
      * [Supported Access Modes per Storage Type](#supported-access-modes-per-storage-type)
      * [Block Volume Support](#block-volume-support)
      * [OpenShift Container Storage](#openshift-container-storage)
         * [Minimum Requirements for OCS nodes](#minimum-requirements-for-ocs-nodes)
* [Installation Process](./INSTALLATION.md)

## Resource Management

You should contain all your OpenShift resources inside a single resource group or project.  This will simplify resource management and cleanup of your OpenShift clusters.  Each cloud provider has a different way of organizing resources:  in Azure and AWS, you use resource groups; in GCP you would organize everything under a project.

## Network

Each cloud provider allows you to create a Virtual Private Cloud (VPC)  or a VNET (Virtual Network) to deploy your infrastrcture components on with enhanced security and isolation.  You should create a different VPC/VNET per cluster, and use different cloud provider constructs to ensure that your cluster and other private networks in your business can talk to each other.

### Subnets

Split control plane and workers into different subnets.  All VMs will be placed in a private subnet inside the VPC or VNET.  They are exposed to the outside world via LoadBalancer endpoints as needed.

### Availability Zones

Spread masters and workers into different AZs in region to ensure redundancy and high availability.

### LoadBalancers

Each LoadBalancer requires a **static** IP address.  If you want to expose your applications to the internet, you need a public IP address.  If you want to expose management of the cluster to the internet, you need a public IP address.  You can choose to keep everything private.

We require at least 3 LoadBalancers to ensure proper operation of your OpenShift cluster

* `api-int.<cluster_name>.<base_domain>` - Used for internal cluster machine configuration (bootstrap) process and any subsequent node provisioning event (ie: worker scaling via MachineSets).  Use an internal IP address.  Needs to load balance port 22623 to all master nodes and bootstrap server.
* `api.<cluster_name>.<base_domain>` - Used for Kubernetes API, management of the cluster via oc and kubectl commands. Can use public or private IP address.  Needs to load balance port 6443 to all master nodes.
* `*.apps.<cluster_name>.<base_domain>` - Used to expose your workloads routes outside of the cluster. Can use public or private IP address

### Security Groups

The following network ports are required for effective operation of your OpenShift cluster.

From node to node:

| Protocol        | Port                                                         | Description                          |
| :-------------- | :----------------------------------------------------------- | :----------------------------------- |
| TCP             | `2379`-`2380`                                                | etcd server, peer, and metrics ports |
|| `6443`          | Kubernetes API                                               |                                      |
|| `9000`-`9999`   | Host level services, including the node exporter on ports `9100`-`9101` and the Cluster Version Operator on port `9099`. |                                      |
|| `10249`-`10259` | The default ports that Kubernetes reserves                   |                                      |
|| `10256`         | openshift-sdn                                                |                                      |
| UDP             | `4789`                                                       | VXLAN and GENEVE                     |
|| `6081`          | VXLAN and GENEVE                                             |                                      |
|| `9000`-`9999`   | Host level services, including the node exporter on ports `9100`-`9101`. |                                      |
|| `30000`-`32767` | Kubernetes NodePort                                          |                                      |

To loadbalancers:

| Port    | Machines                                                     | Internal | External | Description           |
| :------ | :----------------------------------------------------------- | :------: | :------: | :-------------------- |
| `6443`  | Bootstrap and control plane. You remove the bootstrap machine from the load balancer after the bootstrap machine initializes the cluster control plane. |    ✅     |    ✅     | Kubernetes API server |
| `22623` | Bootstrap and control plane. You remove the bootstrap machine from the load balancer after the bootstrap machine initializes the cluster control plane. |    ✅     |          | Machine Config server |
| `443`   | The machines that run the Ingress router pods, compute, or worker, by default. |    ✅     |    ✅     | HTTPS traffic         |
| `80`    | The machines that run the Ingress router pods, compute, or worker by default. |    ✅     |    ✅     | HTTP traffic          |


## DNS

DNS is a critical component of OCP4.  Getting it right is paramount to a successful deployment.  Your cluster should use 2 DNS zones.  This document will use a base domain of `example.com` and a cluster name of `openshift4`

### Private DNS Zone

Create a private DNS zone in your cloud provider called `<cluster_name>.<base_domain>`,  Eg. `openshift4.example.com`. This zone will be used by the internal cluster compute nodes, and should resolve the following records:

1. `etcd-<index>.<cluster_name>.<base_domain>` -OpenShift Container Platform requires DNS A records for each etcd instance to point to the control plane machines that host the instances. The etcd instances are differentiated by `<index>` values, which start with `0` and end with `n-1`, where `n` is the number of control plane machines in the cluster. The DNS record must resolve to an unicast IPv4 address for the control plane machine, and the records must be resolvable from all the nodes in the cluster.

2. `_etcd-server-ssl._tcp.<cluster_name>.<base_domain>` - For each control plane machine, OpenShift Container Platform also requires a SRV DNS record for etcd server on that machine with priority `0`, weight `10` and port `2380`. A cluster that uses three control plane machines requires the following records:

```bash
# _service._proto.name.                            TTL    class SRV priority weight port target.
_etcd-server-ssl._tcp.<cluster_name>.<base_domain>  86400 IN    SRV 0        10     2380 etcd-0.<cluster_name>.<base_domain>.
_etcd-server-ssl._tcp.<cluster_name>.<base_domain>  86400 IN    SRV 0        10     2380 etcd-1.<cluster_name>.<base_domain>.
_etcd-server-ssl._tcp.<cluster_name>.<base_domain>  86400 IN    SRV 0        10     2380 etcd-2.<cluster_name>.<base_domain>.
```

3. `api.<cluster_name>.<base_domain>` - This DNS record must point to the load balancer for the control plane machines. This record must be resolvable by both clients external to the cluster and from all the nodes within the cluster. **The API server must be able to resolve the worker nodes by the host names that are recorded in Kubernetes. If it cannot resolve the node names, proxied API calls can fail, and you cannot retrieve logs from Pods.**

4. `api-int.<cluster_name>.<base_domain>` - This DNS record must point to the internal load balancer for the control plane machines. This record must be resolvable from all the nodes within the cluster. 

5. `*.apps.<cluster_name>.<base_domain>` - A wildcard DNS record that points to the load balancer that targets the machines that run the Ingress router pods, which are the worker nodes by default. This record must be resolvable by both clients external to the cluster and from all the nodes within the cluster.

### Public DNS Zone

Create a public DNS zone in your cloud provider called <base_domain>, Eg. `example.com`.  Delegate this zone to your cloud provider.  This zone will be used by anyone that wants to access your cluster externally, and should resolve the following records

1. `api.<cluster_name>.<base_domain>` - This DNS record must point to the load balancer for the control plane machines. This record must be resolvable by both clients external to the cluster and from all the nodes within the cluster.  It's used for management functions of your cluster, including the use of the `oc` command line utility.
2. `*.apps.<cluster_name>.<base_domain>` - A wildcard DNS record that points to the load balancer that targets the machines that run the Ingress router pods, which are the worker nodes by default. This record must be resolvable by both clients external to the cluster and from all the nodes within the cluster. It's used for Web browser and REST access to your applications.



## Access Management

OpenShift 4.x automates many of the tasks required to deploy a usable cluster:  It will configure your image registry storage on a per-cloud basis, it will create and destroy worker nodes automatically to scale up or down your cluster, and it will deploy a loadbalancer to handle application traffic, as well as add worker nodes to the `*.apps` loadbalancer.  This means we need to pass along cloud credentials with enough permissions to the cloud provider so the cluster operators can handle these automated tasks.



## Cluster Nodes

### Control Plane Nodes (masters)

In a Kubernetes cluster, the master nodes run services that are required to control the Kubernetes cluster. In OpenShift Container Platform, the master machines are the control plane. They contain more than just the Kubernetes services for managing the OpenShift Container Platform cluster. Because all of the machines with the control plane role are master machines, and the terms "master" and "control plane" are used interchangeably to describe them. Instead of being grouped into a MachineSet, master machines are defined by a series of standalone machine API resources. Extra controls apply to master machines to prevent you from deleting all master machines and breaking your cluster.

A special compute node, called the Bootstrap node, is used to initialize the cluster.  This bootstrap node can be removed from the cluster after installation is complete.

Services that fall under the Kubernetes category on the master include the API server, etcd, controller manager server, and HAProxy services. Since these services reflect the core underlying heart of the platform, special attention should be given to the performance characteristics used for these nodes.

You should spread out these control plane nodes across different availability zones in your cloud provider region.

### Compute Nodes (workers)

In a Kubernetes cluster, the worker nodes are where the actual workloads requested by Kubernetes users run and are managed. The worker nodes advertise their capacity and the scheduler, which is part of the master services, determines on which nodes to start containers and Pods. Important services run on each worker node, including CRI-O, which is the container engine, Kubelet, which is the service that accepts and fulfills requests for running and stopping container workloads, and a service proxy, which manages communication for pods across workers.

In OpenShift Container Platform, MachineSets control the worker machines. Machines with the worker role drive compute workloads that are governed by a specific machine pool that autoscales them. Because OpenShift Container Platform has the capacity to support multiple machine types, the worker machines are classed as *compute* machines. In this release, the terms "worker machine" and "compute machine" are used interchangeably because the only default type of compute machine is the worker machine. In future versions of OpenShift Container Platform, different types of compute machines, such as infrastructure machines, might be used by default.

You should spread out these worker nodes across different availability zones in your cloud provider region.

### Recommended Resource Sizing

| Machine       | Count | Operating System | vCPU | RAM  | Storage |
| ------------- | ----- | ---------------- | ---- | ---- | ------- |
| Bootstrap     | 1     | RHCOS            | 4    | 16   | 120 GB  |
| Control plane | 3     | RHCOS            | 8    | 32   | 200 GB  |
| Compute       | 2     | RHCOS            | 8    | 32   | 200 GB  |

Since we can create cluster workers via MachineSets with the performance characteristics we need, its not as critical to define your cluster with the exact performance characteristics at deployment time.  Control plane nodes however, should have enough CPU, memory and disk to meet your performance requirements.

### OS Image

As part of their cloud provisioning strategy, RedHat has made available images that you can use for your RHCOS vms.  They are broken down by cloud provider and, in come cases, regions.  A list of all the images available can be found [here](https://github.com/openshift/installer/blob/master/data/data/rhcos.json).  The list is updated [frequently](https://github.com/openshift/installer/commits/master/data/data/rhcos.json) as new features are added and issues are fixed.  Ensure you're using the latest image avaialble for your cluster.



## Storage

### Compute Node Storage

Special consideration should be given to the storage used for the OS disk of your control plane virtual machines.  You should use the fastest disk type available on your cloud provider, usually SSDs or NVMe.  You can use slower disk types for your master nodes, but performance might suffer due to etcd sync issues across master nodes.

### Persistent Volume for Image Registry

If you're deploying across any major cloud provider, the openshift-image-registry operator will create and manage for you a PVC for your image registry.  You do not need to do anything as long as the credentials you've passed to your cluster have proper permissions to create storage infrastructure in your cloud provider.

### Persistent Volumes for Workloads

For you workloads, you can also leverage your cloud provider storage classes.  A PersistentVolume can be mounted on a host in any way supported by the resource provider. Providers have different capabilities and each PV’s access modes are set to the specific modes supported by that particular volume.

Block storage is suitable for databases and other low-latency transactional workloads. Some examples of supported workloads are Red Hat OpenShift Container Platform logging and monitoring, and PostgreSQL.

Object storage is for video and audio files, compressed data archives, and the data used to train artificial intelligence or machine learning programs. In addition, object storage can be used for any application developed with a cloud-first approach.

File storage is for continuous integration and delivery, web application file storage, and artificial intelligence or machine learning data aggregation.

Take a look at the tables bellow to decide which storage class best fits your needs.

#### Access Modes
The following table lists the access modes:

| Access Mode   | CLI abbreviation | Description                                               |
| :------------ | :--------------- | :-------------------------------------------------------- |
| ReadWriteOnce | RWO              | The volume can be mounted as read-write by a single node. |
| ReadOnlyMany  | ROX              | The volume can be mounted as read-only by many nodes.     |
| ReadWriteMany | RWX              | The volume can be mounded as read-write by many nodes.    |

#### Supported Access Modes per Storage Type
The following table lists supported access modes for PVs

| Volume Plug-in                      | ReadWriteOnce | ReadOnlyMany | ReadWriteMany |
| :---------------------------------- | :-----------: | :----------: | :-----------: |
| AWS EFS                             | ✅            | ✅          | ✅            |
| AWS EBS                             | ✅            | ⛔️        | ⛔️           |
| Azure File                          | ✅            | ✅          | ✅           |
| Azure Disk                          | ✅            | ⛔️         | ⛔️           |
| Cinder                              | ✅            | ⛔️         | ⛔️           |
| Fibre Channel                       | ✅            | ✅          | ⛔️           |
| GCE Persistent Disk                 | ✅            | ⛔️         | ⛔️           |
| HostPath                            | ✅            | ⛔️         | ⛔️           |
| iSCSI                               | ✅            | ✅          | ⛔️           |
| Local volume                        | ✅            | ⛔️         | ⛔️           |
| NFS                                 | ✅            | ✅          | ✅           |
| Red Hat OpenShift Container Storage | ✅            | ⛔️         | ✅           |
| VMware vSphere                      | ✅            | ⛔️         | ⛔️          |

#### Block Volume Support
OpenShift Container Platform can statically provision raw block volumes. These volumes do not have a file system, and can provide performance benefits for applications that either write to the disk directly or implement their own storage service.

| Volume Plug-in                      | Manually provisioned | Dynamically provisioned | Fully supported |
| :---------------------------------- | :-----------: | :----------: | :-----------: |
| AWS EBS                             | ✅            | ✅          | ✅            |
| Azure File                          | ⛔️          | ⛔️        | ⛔️          |
| Azure Disk                          | ✅            | ✅           | ✅             |
| Cinder                              | ⛔️          | ⛔️         | ⛔️           |
| Fibre Channel                       | ✅            | ⛔️         | ⛔️           |
| GCP                                 | ✅            | ✅          | ✅            |
| HostPath                            | ⛔️          | ⛔️         | ⛔️           |
| iSCSI                               | ✅            | ⛔️         | ⛔️           |
| Local volume                        | ✅            | ⛔️         | ✅            |
| NFS                                 | ⛔️          | ⛔️        | ⛔️          |
| Red Hat OpenShift Container Storage | ✅            | ✅          | ✅           |
| VMware vSphere                      | ✅            | ✅          | ✅           |

#### OpenShift Container Storage

Red Hat [OpenShift Container Storage](https://www.openshift.com/products/container-storage/) is software-defined storage integrated with and optimized for Red Hat OpenShift Container Platform. OpenShift Container Storage 4.2 is built on Red Hat Ceph® Storage, Rook, and NooBaa to provide container native storage services that support block, file, and object services. For the initial 4.2 release, OpenShift Container Storage will be supported on OpenShift platforms deployed on Amazon Web Services and VMware. It will install anywhere OpenShift does: on-premise or in the public cloud.

Leveraging the Kubernetes Operator framework, OpenShift Container Storage (OCS) automates a lot of the complexity involved in providing cloud native storage for OpenShift. OCS integrates deeply into cloud native environments by providing a seamless experience for scheduling, lifecycle management, resource management, security, monitoring, and user experience of the storage resources.

To deploy OpenShift Container Storage, the administrator can go to the OpenShift Administrator Console and navigate to the [OperatorHub](https://operatorhub.io/) to find the OpenShift Container Storage Operator.  During installation, you will be prompted to select the nodes that will be used to setup the storage cluster.  While you can use the openshift worker nodes to install OCS on, it is recommended that you create additional MachineSet to deploy your storage cluster.

##### Minimum Requirements for OCS nodes

| Count| vCPU | Memory (GiB) | Storage |
| ---- | ---- | ------------ | ------- |
| 3    | 16   | 64           | 2TB     |



## [Installation Process](./INSTALLATION.md)

