# Lab: Persistent Volume Claims

Coming soon.

## Prerequisites

* Complete previous labs for [AKS](../../create-aks-cluster/README.md) and [ACR](../../build-application/README.md).

## Instructions

A persistent volume represents a piece of storage that has been provisioned for use with Kubernetes pods. A persistent volume can be used by one or many pods and can be dynamically or statically provisioned. For more information on Kubernetes persistent volumes, see [Kubernetes persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). This article shows you how to use persistent volumes with Azure disks in an Azure Kubernetes Service (AKS) cluster.

Note

An Azure disk can only be mounted with *Access mode* type *ReadWriteOnce*, which makes it available to only a single AKS node. If needing to share a persistent volume across multiple nodes, consider using [Azure Files](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv).

Built in storage classes[](https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv#built-in-storage-classes)
----------------------------------------------------------------------------------------------------------------------

A storage class is used to define how a unit of storage is dynamically created with a persistent volume. For more information on Kubernetes storage classes, see [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/).

Each AKS cluster includes two pre-created storage classes, both configured to work with Azure disks:

-   The *default* storage class provisions a standard Azure disk.
    -   Standard storage is backed by HDDs, and delivers cost-effective storage while still being performant. Standard disks are ideal for a cost effective dev and test workload.
-   The *managed-premium* storage class provisions a premium Azure disk.
    -   Premium disks are backed by SSD-based high-performance, low-latency disk. Perfect for VMs running production workload. If the AKS nodes in your cluster use premium storage, select the *managed-premium* class.

Use the [kubectl get sc](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get) command to see the pre-created storage classes. The following example shows the pre-create storage classes available within an AKS cluster:

Copy

```
$ kubectl get sc

NAME                PROVISIONER                AGE
default (default)   kubernetes.io/azure-disk   1h
managed-premium     kubernetes.io/azure-disk   1h

```

Note

Persistent volume claims are specified in GiB but Azure managed disks are billed by SKU for a specific size. These SKUs range from 32GiB for S4 or P4 disks to 4TiB for S50 or P50 disks. The throughput and IOPS performance of a Premium managed disk depends on the both the SKU and the instance size of the nodes in the AKS cluster. For more information, see [Pricing and Performance of Managed Disks](https://azure.microsoft.com/pricing/details/managed-disks/).

Create a persistent volume claim[](https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv#create-a-persistent-volume-claim)
--------------------------------------------------------------------------------------------------------------------------------------

A persistent volume claim (PVC) is used to automatically provision storage based on a storage class. In this case, a PVC can use one of the pre-created storage classes to create a standard or premium Azure managed disk.

Create a file named `azure-premium.yaml`, and copy in the following manifest. The claim requests a disk named `azure-managed-disk` that is *5GB* in size with *ReadWriteOnce* access. The *managed-premium* storage class is specified as the storage class.

yamlCopy

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-managed-disk
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 5Gi

```

Tip

To create a disk that uses standard storage, use `storageClassName: default` rather than *managed-premium*.

Create the persistent volume claim with the [kubectl create](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create) command and specify your *azure-premium.yaml*file:

Copy

```
$ kubectl create -f azure-premium.yaml

persistentvolumeclaim/azure-managed-disk created

```

Use the persistent volume[](https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv#use-the-persistent-volume)
------------------------------------------------------------------------------------------------------------------------

Once the persistent volume claim has been created and the disk successfully provisioned, a pod can be created with access to the disk. The following manifest creates a basic NGINX pod that uses the persistent volume claim named *azure-managed-disk* to mount the Azure disk at the path `/mnt/azure`.

Create a file named `azure-pvc-disk.yaml`, and copy in the following manifest.

yamlCopy

```
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/mnt/azure"
        name: volume
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: azure-managed-disk

```

Create the pod with the [kubectl create](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create) command, as shown in the following example:

Copy

```
$ kubectl create -f azure-pvc-disk.yaml

pod/mypod created

```

You now have a running pod with your Azure disk mounted in the `/mnt/azure` directory. This configuration can be seen when inspecting your pod via `kubectl describe pod mypod`, as shown in the following condensed example:

Copy

```
$ kubectl describe pod mypod

[...]
Volumes:
  volume:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  azure-managed-disk
    ReadOnly:   false
  default-token-smm2n:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-smm2n
    Optional:    false

Events:
  Type    Reason                 Age   From                               Message
  ----    ------                 ----  ----                               -------
  Normal  Scheduled              2m    default-scheduler                  Successfully assigned mypod to aks-nodepool1-79590246-0
  Normal  SuccessfulMountVolume  2m    kubelet, aks-nodepool1-79590246-0  MountVolume.SetUp succeeded for volume "default-token-smm2n"
  Normal  SuccessfulMountVolume  1m    kubelet, aks-nodepool1-79590246-0  MountVolume.SetUp succeeded for volume "pvc-faf0f176-8b8d-11e8-923b-deb28c58d242"
[...]

```

Back up a persistent volume[](https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv#back-up-a-persistent-volume)
----------------------------------------------------------------------------------------------------------------------------

To back up the data in your persistent volume, take a snapshot of the managed disk for the volume. You can then use this snapshot to create a restored disk and attach to pods as a means of restoring the data.

First, get the volume name with the `kubectl get pvc` command, such as for the PVC named *azure-managed-disk*:

Copy

```
$ kubectl get pvc azure-managed-disk

NAME                 STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
azure-managed-disk   Bound     pvc-faf0f176-8b8d-11e8-923b-deb28c58d242   5Gi        RWO            managed-premium   3m

```

This volume name forms the underlying Azure disk name. Query for the disk ID with [az disk list](https://docs.microsoft.com/cli/azure/disk#az-disk-list) and provide your PVC volume name, as shown in the following example:

Copy

```
$ az disk list --query '[].id | [?contains(@,`pvc-faf0f176-8b8d-11e8-923b-deb28c58d242`)]' -o tsv

/subscriptions/<guid>/resourceGroups/MC_MYRESOURCEGROUP_MYAKSCLUSTER_EASTUS/providers/MicrosoftCompute/disks/kubernetes-dynamic-pvc-faf0f176-8b8d-11e8-923b-deb28c58d242

```

Use the disk ID to create a snapshot disk with [az snapshot create](https://docs.microsoft.com/cli/azure/snapshot#az-snapshot-create). The following example creates a snapshot named *pvcSnapshot* in the same resource group as the AKS cluster (*MC_myResourceGroup_myAKSCluster_eastus*). You may encounter permission issues if you create snapshots and restore disks in resource groups that the AKS cluster does not have access to.

Azure CLICopy

```
$ az snapshot create\
    --resource-group MC_myResourceGroup_myAKSCluster_eastus\
    --name pvcSnapshot\
    --source /subscriptions/<guid>/resourceGroups/MC_myResourceGroup_myAKSCluster_eastus/providers/MicrosoftCompute/disks/kubernetes-dynamic-pvc-faf0f176-8b8d-11e8-923b-deb28c58d242

```

Depending on the amount of data on your disk, it may take a few minutes to create the snapshot.

Restore and use a snapshot[](https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv#restore-and-use-a-snapshot)
--------------------------------------------------------------------------------------------------------------------------

To restore the disk and use it with a Kubernetes pod, use the snapshot as a source when you create a disk with [az disk create](https://docs.microsoft.com/cli/azure/disk#az-disk-create). This operation preserves the original resource if you then need to access the original data snapshot. The following example creates a disk named *pvcRestored* from the snapshot named *pvcSnapshot*:

Azure CLICopy

```
az disk create --resource-group MC_myResourceGroup_myAKSCluster_eastus --name pvcRestored --source pvcSnapshot

```

To use the restored disk with a pod, specify the ID of the disk in the manifest. Get the disk ID with the [az disk show](https://docs.microsoft.com/cli/azure/disk#az-disk-show) command. The following example gets the disk ID for *pvcRestored* created in the previous step:

Azure CLICopy

```
az disk show --resource-group MC_myResourceGroup_myAKSCluster_eastus --name pvcRestored --query id -o tsv

```

Create a pod manifest named `azure-restored.yaml` and specify the disk URI obtained in the previous step. The following example creates a basic NGINX web server, with the restored disk mounted as a volume at */mnt/azure*:

yamlCopy

```
kind: Pod
apiVersion: v1
metadata:
  name: mypodrestored
spec:
  containers:
    - name: myfrontendrestored
      image: nginx
      volumeMounts:
      - mountPath: "/mnt/azure"
        name: volume
  volumes:
    - name: volume
      azureDisk:
        kind: Managed
        diskName: pvcRestored
        diskURI: /subscriptions/<guid>/resourceGroups/MC_myResourceGroupAKS_myAKSCluster_eastus/providers/Microsoft.Compute/disks/pvcRestored

```

Create the pod with the [kubectl create](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create) command, as shown in the following example:

Copy

```
$ kubectl create -f azure-restored.yaml

pod/mypodrestored created

```

You can use `kubectl describe pod mypodrestored` to view details of the pod, such as the following condensed example that shows the volume information:

Copy

```
$ kubectl describe pod mypodrestored

[...]
Volumes:
  volume:
    Type:         AzureDisk (an Azure Data Disk mount on the host and bind mount to the pod)
    DiskName:     pvcRestored
    DiskURI:      /subscriptions/19da35d3-9a1a-4f3b-9b9c-3c56ef409565/resourceGroups/MC_myResourceGroupAKS_myAKSCluster_eastus/providers/Microsoft.Compute/disks/pvcRestored
    Kind:         Managed
    FSType:       ext4
    CachingMode:  ReadWrite
    ReadOnly:     false
[...]
```

## Troubleshooting / Debugging

* ?

## Docs / References

* ?