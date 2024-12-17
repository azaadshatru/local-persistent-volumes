# Create Local Persistent Volumes for OpenSearch
Deploying OpenSearch requires a StorageClass that provides persistent block storage or a local file system mount in order to store the search indices. Remote file systems, such as NFS, are not supported for this purpose. However, the default storage class in cloud platforms is typically adequate. In case you are deploying SAS Viya on OpenShift or OpenSource platforms, you need local storage.
AS per SAS Official Documentation: "On cloud or virtualization platforms, SAS recommends provisioning the required storage using features that are provided by the platform, such as Azure Disk or vSphere volumes. For physical machines or for virtualization platforms that lack dynamic provisioning, SAS recommends using local storage: persistent volumes with disks of sufficient size and a Kubernetes provisioner for local storage."
So we need to create Local Persistent Volumes using Static Provisioner. This will enable the use of the local persistent volumes for stateful workloads on SAS Viya deployments specially OpenSearch.

## High level steps to perform

* A discovery directory named /openSearch, or anything as per your convenience, to contain mounts for storage to provide to stateful resources
* A storage class named local-storage which can be used by stateful resources to access local storage
* The static provisioner deployed in the local-provisioner namespace to manage the volumes.

## Pre-Requirements
The following are the pre-requisites before we begin:
  - Kubernetes cluster has already been deployed.
  - Identify the Kubernetes nodes for stateful workloads and attached the additional disks for storage.
  - define the topology to deploy
      - In our case we are using 3 replicas of OpenSearch - 3 Master Nodes and 3 Data Nodes - So we need total 6 volumes.
      - No of stateful nodes, storage capacity and size will be provided to you in the Sizing document.

## Step 1 - Mount the disk
On the K8s workers nodes which are identified/labeled/tainted as Stateful Nodes we need to mount the additional disks we added. In this example we are using 3 Stateful nodes.
You need to take help of your Unix-Linux Team with root previleges.
```
On node 1:
mkdir /openSearch/volume_1_node_1
mkdir /openSearch/volume_2_node_1
On node 2:
mkdir /openSearch/volume_1_node_2
mkdir /openSearch/volume_2_node_2
On node 3:
mkdir /openSearch/volume_1_node_3
mkdir /openSearch/volume_2_node_3
```
Mount each additional disk attached on each node to the dir defined in the example above
```
Ask your Unix Team to format the addional disk and create volumes of it and mount the same on folders created like /openSearch/volume_1_node_1
```
![image](https://github.com/user-attachments/assets/00ae713f-b575-4879-9189-b9eb23c473f9)

To make this mount persistent, make the entry inside /etc/fstab file

```

