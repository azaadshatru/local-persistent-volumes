# Create Local Persistent Volumes for OpenSearch
Deploying OpenSearch requires a StorageClass that provides persistent block storage or a local file system mount in order to store the search indices. Remote file systems, such as NFS, are not supported for this purpose. However, the default storage class in cloud platforms is typically adequate. In case you are deploying SAS Viya on OpenShift or OpenSource platforms, you need local storage. <br />

AS per SAS Official Documentation: "On cloud or virtualization platforms, SAS recommends provisioning the required storage using features that are provided by the platform, such as Azure Disk or vSphere volumes. For physical machines or for virtualization platforms that lack dynamic provisioning, SAS recommends using local storage: persistent volumes with disks of sufficient size and a Kubernetes provisioner for local storage."<br />

So we need to create Local Persistent Volumes using Static Provisioner. This will enable the use of the local persistent volumes for stateful workloads on SAS Viya deployments specially OpenSearch.

### Important Note: These steps need to considered during the first deployment of the SAS Viya 4 environment. After you have completed the deployment process for the SAS Viya platform, you cannot change the OpenSearch deployment type without a full redeployment.

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
Mount each additional disk attached on each node to the dir defined in the example above <br />

Ask your Unix Team to format the addional disk attached to the 3 stateful nodes and create volumes out of it, mount the same on folders created like /openSearch/volume_1_node_1 .<br/>
Run lsblk command and see the volumes created: In our example we have /dev/sdd and /dev/sdc, 2 disks attached on node 1 which are mounted on /openSearch/volume_1_node_1 and /openSearch/volume_2_node_1

![image](https://github.com/user-attachments/assets/62b55345-3aa9-4e1a-a7af-ccc526cda5fc)

To make these mount persistent, make the entry inside /etc/fstab file
![image](https://github.com/user-attachments/assets/27c47e63-b8d0-4ea7-835b-02329b8d3c54)

## Step 2 - Create Storage Class
Create a local-provisioner namespace to deployment the provisioner to.
```
kubectl create namespace local-provisioner
```
Create a storage class named local-storage to use for mounting the volumes created in /openSearch
```
kubectl apply -f storage_class.yaml
```
| storage_class.yaml                         |
|:-------------------------------------------|
```
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
# Supported policies: Delete, Retain
reclaimPolicy: Delete
#allowVolumeExpansion: true
```
After applying the above yaml file, you will observe new storage being created with the name local-storage
```
kubectl get sc
```
![image](https://github.com/user-attachments/assets/6c32667c-0a88-41e8-8d77-53dcf4331193)


Deploy the provisioner by applying the following YAML file.
```
kubectl -n local-provisioner apply -f local_storage_provisioner.yaml
```
| local_storage_provisioner.yaml             |
|:-------------------------------------------|
```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-provisioner-config
  namespace: local-provisioner
data:
  storageClassMap: |  
    local-storage:
       hostDir: /openSearch
       mountDir:  /openSearch
       blockCleanerCommand:
         - "/scripts/shred.sh"
         - "2"
       volumeMode: Filesystem
       fsType: xfs
       namePattern: "*"
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: local-volume-provisioner
  namespace: local-provisioner
  labels:
    app: local-volume-provisioner
spec:
  selector:
    matchLabels:
      app: local-volume-provisioner
  template:
    metadata:
      labels:
        app: local-volume-provisioner
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload.sas.com/class
                operator: In
                values:
                - stateful
              matchFields: []
      serviceAccountName: local-storage-admin
      containers:
        - image: "registry.k8s.io/sig-storage/local-volume-provisioner:v2.4.0"
          imagePullPolicy: "IfNotPresent"
          name: provisioner
          securityContext:
            privileged: true
          env:
          - name: MY_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          volumeMounts:
            - mountPath: /etc/provisioner/config
              name: provisioner-config
              readOnly: true
            - name: provisioner-dev
              mountPath: /dev
            - mountPath:  /openSearch
              name: local-storage
              mountPropagation: "HostToContainer"
      tolerations:
      - effect: NoSchedule
        key: workload.sas.com/class
        operator: Equal
        value: stateful
      volumes:
        - name: provisioner-config
          configMap:
            name: local-provisioner-config
        - name: provisioner-dev
          hostPath:
            path: /dev
        - name: local-storage
          hostPath:
            path: /openSearch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: local-storage-admin
  namespace: local-provisioner
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: local-storage-provisioner-pv-binding
  namespace: local-provisioner
subjects:
- kind: ServiceAccount
  name: local-storage-admin
  namespace: local-provisioner
roleRef:
  kind: ClusterRole
  name: system:persistent-volume-provisioner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: local-storage-provisioner-node-clusterrole
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: local-storage-provisioner-node-binding
  namespace: local-provisioner
subjects:
- kind: ServiceAccount
  name: local-storage-admin
  namespace: local-provisioner
roleRef:
  kind: ClusterRole
  name: local-storage-provisioner-node-clusterrole
  apiGroup: rbac.authorization.k8s.io

```
Once you apply the above yaml file, you will that 6 new PVs are created
```
kubectl get pv | grep local
```
![image](https://github.com/user-attachments/assets/39848a72-0a77-4263-8349-42f30e2e52f3)

You can also observe the 3 pods running on 3 stateful nodes each under local-provisioner namespace
```
kubectl get pods -n local-provisioner -o wide
```
![image](https://github.com/user-attachments/assets/23af9e2e-3a7d-40e5-8e7f-e839d922125e)

# Configure OpenSearch to use Local Persistent Volumes
Once you have deployed the local-storage storage class and created PVs using the local provisioner successfully, you can now configure OpenSearch to use these local persistent volumes and configure the topology as per our example to user 3 custom master nodes and 3 custom data nodes as per the instructions below:<br/>

Copy the sample all files/folders from configure-elasticsearch folder from sas-bases/example folder inside site-config folder

```
cp -r $deploy/sas-bases/example/configure-elasticsearch $deploy/site-config/.
```
Edit storage-class-transformer.yaml file $deploy/site-config/configure-elasticsearch/internal/storage/storage-class-transformer.yaml and change the value of STORAGE CLASS to your local-storage storage class

From this...
```
---
apiVersion: builtin
kind: PatchTransformer
metadata:
  name: sas-opendistro-storage-class-transformer
patch: |-
  - op: replace
    path: /spec/defaultStorageClassName
    value: {{ STORAGE CLASS }}
target:
  kind: OpenDistroCluster
```
To this...
```
---
apiVersion: builtin
kind: PatchTransformer
metadata:
  name: sas-opendistro-storage-class-transformer
patch: |-
  - op: replace
    path: /spec/defaultStorageClassName
    value: local-storage
target:
  kind: OpenDistroCluster
```

Edit custom-topology.yaml file $deploy/site-config/configure-elasticsearch/internal/topology/custom-topology.yaml 
```
---
apiVersion: builtin
kind: PatchTransformer
metadata:
  name: sas-opendistro-custom-topology
# update the patch value to define cluster nodes 
patch: |-
  - op: replace
    path: /spec/nodes
    value:
      - name: custom-master
        replicas: 3
        roles:
          - master
        heapsize: 2G
      - name: custom-data
        replicas: 3
        roles:
          - data
        heapsize: 8G

  # note that additional op replace blocks can be used to alter image, env, jvm, security and other paths      

target:
  kind: OpenDistroCluster
  name: sas-opendistro

```
Add above two transformers to the root kustomization.yaml file.
```
transformers:
# Existing entries here.
- site-config/configure-elasticsearch/internal/storage/storage-class-transformer.yaml
- site-config/configure-elasticsearch/internal/topology/custom-topology.yaml
```

You can now deploy the SAS Viya 4 environment using the following commands
```
kubectl -n <name of namespace> apply --selector="sas.com/admin=cluster-api" --server-side --force-conflicts -f site.yaml
kubectl -n <name of namespace> wait --for condition=established --timeout=60s -l "sas.com/admin=cluster-api" crd
kubectl -n <name of namespace> apply --selector="sas.com/admin=cluster-wide" -f site.yaml
kubectl -n <name of namespace> apply --selector="sas.com/admin=cluster-local" -f site.yaml --prune
kubectl -n <name of namespace> apply --selector="sas.com/admin=namespace" -f site.yaml --prune
```

Once deployment is completed you will see 6 pods running 
```
kubectl get pods | grep open
```
![image](https://github.com/user-attachments/assets/fc81b081-6f8e-43f2-9562-a0fea254d6c3)

You can also observe new 6 PVCs created
```
kubectl get pvc | grep open
```
![image](https://github.com/user-attachments/assets/e05ca7a8-222f-4df5-80dc-a9bd1405c7c8)

The PVs which were earlier in available state are now Bound to newly created PVCs
```
kubectl get pv | grep local
```
![image](https://github.com/user-attachments/assets/fabbcdc7-0067-4a46-9ed8-a139bf72055f)

Described one of the PVs to show that they are using the local volume from one of the stateful node
```
kubectl descrive pv local-pv-63c00f08
```
![image](https://github.com/user-attachments/assets/e34fe32e-8435-485a-8e9f-07728bf93e33)

This marks the successful completion of configuration of local storage for OpenSearch.


