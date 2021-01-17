---
layout: post 
title: Kubernetes PersistentVolume Subsystem
tags: [kubernetes, pv, pvc]
---

Container storage is `ephemeral`, it goes away when the container does. 
Because of this, storage needs to be independent of the container in order to live beyond the container. 
In Kubernetes, `volumes` provide the abstraction to decouple storage from the pod's containers. 
When we attach a volume to a pod it provides a directory mounted inside the pod's containers so that we can access files.
There are many different volume types.

### emptyDir

The [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) volume type is the simplest. It is backed by the host currently running the pod. When the pod is scheduled on the node, it creates an empty directory on the node.
It is not completely permanent, if the pod is removed from the node and moved to another node, the data is deleted. However, it will persist beyond the life of the container. 
It allows multiple containers within a pod to have both read and write access to the files in the `emptyDir` volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: busybox1
      image: busybox
      command: [ "/bin/sh", "-c", "while true; do sleep 3600; done" ]
      volumeMounts:
        - mountPath: /data
          name: data
    - name: busybox2
      image: busybox
      command: [ "/bin/sh", "-c", "while true; do sleep 3600; done" ]
      volumeMounts:
        - mountPath: /data
          name: data
  volumes:
    - name: data
      emptyDir: { }
```

After creating the above pod in our cluster, we can validate that a file created with the first container can be accessed by the second container.

```bash
$ kubectl exec -it my-pod -c busybox1 -- /bin/sh
$ cd /data
$ echo "hello" > hello.txt
$ kubectl exec -it my-pod -c busybox2 -- /bin/sh
$ cat /data/hello.txt
hello
```

### PersistentVolume and PersistentVolumeClaim

`PersistentVolume` or `PV` represents a storage resource. If a `Node` in Kubernetes represents CPU and memory resources for a pod, a `PersistentVolume` represents storage resource to a pod.
A `PersistentVolumeClaim` or `PVC` is an abstraction between the pod and the PV. When you create your pod you don't need to worry about where the storage is located, how is implemented, only you need to specify the `PVC` with the required `storageClass` and `accessModes`. (more about these later) 

First create our local Kubernetes cluster. With [kind](https://kind.sigs.k8s.io/) is very easy:

```bash
$ kind delete cluster --name kubernetes-storage-demo
```

Now we are ready to create a `PV` and consume that storage resource within a pod.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  storageClassName: localdisk
  hostPath:
    path: /mnt/data
```
The `hostPath` will allocate storage on an actual node in the cluster where the pod is running.
After creating the `PV` in our cluster, we can see that its status is `Available`. 

```bash
$ kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
mysql-pv   1Gi        RWO            Retain           Available           localdisk               3s
```

Next we create a `PVC`.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: localdisk
  resources:
    requests:
      storage: 500Mi
```

After creating the `PVC` in our cluster, we can see that both the status of the `PV` and `PVC` is BOUND.

```bash
$ kubectl get pvc
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc   Bound    mysql-pv   1Gi        RWO            localdisk      7s
$ kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
mysql-pv   1Gi        RWO            Retain           Bound    default/mysql-pvc   localdisk               91s
```

Finally, we create a pod referencing the `PVC`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  containers:
    - name: mysql
      image: mysql:5.6
      ports:
        - containerPort: 3306
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
      volumeMounts:
        - name: mysql-volume
          mountPath: /var/lib/mysql
  volumes:
    - name: mysql-volume
      persistentVolumeClaim:
        claimName: mysql-pvc
```

After creating the pod in our cluster, we can verify that the storage was created on the Node. With our one node Kubernetes cluster is easy:

```bash
$ docker exec -it kubernetes-storage-demo-control-plane du -sh /mnt/data/mysql
7.0M	/mnt/data/mysql
```

In order to delete our local 1 node Kubernetes cluster we can use the following command.

```bash
$ kind delete cluster --name kubernetes-storage-demo
```

### gcePersistentDisk

Since pods come and go, they are scheduled on different nodes, we need a more robust solution for our data persistence. In this section we are going to use a GKE cluster and delegate the storage to a `PersistentDisk`. 

First lets create a GKE cluster:

```bash
$ gcloud container clusters create kubernetes-storage-demo-cluster
$ kubectl get nodes
gke-kubernetes-storage-d-default-pool-378cb5bc-5tq5   Ready    <none>   7m14s   v1.16.15-gke.6000
gke-kubernetes-storage-d-default-pool-378cb5bc-dq78   Ready    <none>   7m14s   v1.16.15-gke.6000
gke-kubernetes-storage-d-default-pool-378cb5bc-z1dm   Ready    <none>   7m14s   v1.16.15-gke.6000
```

Next we create a GCE Persistent Disk:

```bash
$ gcloud compute disks create --size=10Gi --zone=europe-west6-a mongodb
```

We create a `PV` using the previously created `PersistentDisk` and reference it using `gcePersistentDisk`. 

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  storageClassName: ssd-disk
  gcePersistentDisk:
    pdName: mongodb
```

After this object is created in our cluster we have a 1Gi `PersistentVolume` available in our cluster for consumption that will outlive the lifecycle of any pod that use it.
To consume the volume we create a `PVC`.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ssd-disk
  resources:
    requests:
      storage: 500Mi
```

Kubernetes will match our claim to an available `PV` that meets our requirements (`accessMode`, `storageClassName`, `resource.requests.storage`). For example if the claim is asking for more storage capacity than the `PV`, it will not bind and it will sit there pending. Once a claim is bound, nothing else can claim the same `PV` object unless we free it up.

Finally, we need to consume the `PVC` in our pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
    - name: mongo
      image: mongo
      ports:
        - containerPort: 27017
      volumeMounts:
        - name: mongodb-persitent-storage
          mountPath: /data/db
  volumes:
    - name: mongodb-persitent-storage
      persistentVolumeClaim:
        claimName: mongodb-pvc
```

After the pod is created on the cluster, let's create some data, for example a document using the mongodb shell:

```bash
$ kubectl exec -it mongodb -- mongo 
> use mydb
switched to db mydb
> db.customers.insert({name:'John Doe'})
WriteResult({ "nInserted" : 1 })
> db.customers.find()
{ "_id" : ObjectId("6002be707b7ff91b50cb9fe7"), "name" : "John Doe" }
```

We verify on which node is the pod is running:

```bash
$ kubectl get pods -o wide
NAME      READY   STATUS    RESTARTS   AGE     IP           NODE                                                  NOMINATED NODE   READINESS GATES
mongodb   1/1     Running   0          3m11s   10.100.1.8   gke-kubernetes-storage-d-default-pool-378cb5bc-dq78   <none>           <none>
```

Next we would like to see if the pod is scheduled on another node, we can still access the data. For this first we mark the node as `unschedulable` using

```bash
$ kubectl cordon gke-kubernetes-storage-d-default-pool-378cb5bc-dq78
```

Then we delete and recreate the pod.

```bash
$ kubectl delete -f mongodb-pod.yaml
$ kubectl create -f mongodb-pod.yaml
```

Next, we verify that the pod is running on a different node now:

```bash
$ kubectl get pods -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE                                                  NOMINATED NODE   READINESS GATES
mongodb   1/1     Running   0          52s   10.100.0.3   gke-kubernetes-storage-d-default-pool-378cb5bc-5tq5   <none>           <none>
```

Finally, we see that we still have our data:

```bash
$ kubectl exec -it mongodb -- mongo 
> use mydb
switched to db mydb
> db.customers.find()
{ "_id" : ObjectId("600361e62ba042906cb98af1"), "name" : "John Doe" }
```

### AccessModes

There are 3 ways a pod can access a volume. Not all volumes support all these access modes, generally block based volumes support `ReadWriteOnce` and `ReadOnlyMany`, and file based volumes can support even `ReadWriteMany` access mode. The `PersistentVolume` in GCP does not support for example `ReadWriteMany` access mode. A single volume can only be opened in one mode at a time. 

1. `ReadWriteOnce` (RWO) - can only be mounted as read-write by one pod in the cluster.  
2. `ReadOnlyMany` (ROM) - many pods can mount it, but only in read only mode.
3. `ReadWriteMany` (RWM) - many pods can mount it in read write mode

Worth to note that all replicas included in a `Deployment` will share the same `PVC`. The only way we can support this is by keeping the volume in `ReadOnlyMany` access mode. Even if there is a single pod in our deployment with `ReadWriteOnce` can be problematic depending on the rollout strategy to update a deployment. An updated replica set is not going to be able to mount a volume in read-write mode while the previous replica set still exists.   

### Reclaim Policy

A reclaim policy specifies what happens when a claim on a volume is released. It can be `Delete`(default) or `Retain`. In case of `Delete` when the claim is released the data is removed, however with `Retain` when we can keep the volume and its content, when the claim is released, or even when the pod is failed or deleted.    

### StorageClasses

In the previous example we needed first to create manually the `PersistentDisk` in and create also the `PV` in order to use reference it in our `PVC`. Luckily storage classes make this way more dynamic.  
`StorageClasses` enable dynamic provisioning of volumes. Like everything in Kubernetes, a storage class is an API resource defined in a YAML file:

```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```
The [provisioner](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner) describers what volume plugin is used for provisioning PVs.
In the above example the storage class will provision a zonal sdd disk with ext4 filesystem type.     

Our GKE cluster has already a default storage class:

```bash
$ kubectl get sc
NAME                 PROVISIONER            AGE
standard (default)   kubernetes.io/gce-pd   91s
```

which is using non sdd disk 

```bash
$ kubectl describe sc standard
Name:                  standard
IsDefaultClass:        Yes
Annotations:           storageclass.kubernetes.io/is-default-class=true
Provisioner:           kubernetes.io/gce-pd
Parameters:            type=pd-standard
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>
```

After creating the `fast` StorageClass in our GKE cluster let's use it in PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 1Gi
```

After creating the `mongodb-pvc` we can see the that the status is `BOUND` as opposed to `AVAILABLE` and is using our `fast` storage class. 

```bash
$ kubectl get pvc
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-pvc   Bound    pvc-c0719ec2-9fbd-4d56-8650-7bba6890fabc   1Gi        RWO            fast           3s
```

Behind the scene we can see that both the `PersistentVolume` and the underlying `GCP Persistent Disk` were created:

```bash
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
pvc-c0719ec2-9fbd-4d56-8650-7bba6890fabc   1Gi        RWO            Delete           Bound    default/mongodb-pvc   fast                    4m30s
```

```bash
$ gcloud compute disks list
gke-kubernetes-storage-pvc-c0719ec2-9fbd-4d56-8650-7bba6890fabc  europe-west6-a  zone            1        pd-ssd       READY
```

Storage can be a pretty dry topic, but hopefully with these examples (which you can also try) it is easier to understand. All the examples used in this blog post can be found in my github [repo](https://github.com/altfatterz/learning-kubernetes/tree/master/volume-demo). 



