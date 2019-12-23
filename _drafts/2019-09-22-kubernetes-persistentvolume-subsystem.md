

### gcloud connection setup

Create a new `gcloud` configuration via:
 
```bash
$ gcloud init
```

```bash
$ gcloud config list

[compute]
region = europe-west6
zone = europe-west6-a
[core]
account = cloudnativecoachgcpuser@gmail.com
disable_usage_reporting = True
project = study-pv-subsystem
```

You will be redirected to https://cloud.google.com/sdk/auth_success

```bash
$ gcloud config configurations list

NAME                IS_ACTIVE  ACCOUNT                            PROJECT             DEFAULT_ZONE    DEFAULT_REGION
learning-gcp        False      cloudnativecoachgcpuser@gmail.com  altfatterz          europe-west6-a  europe-west6
study-pv-subsystem  True       cloudnativecoachgcpuser@gmail.com  study-pv-subsystem  europe-west6-a  europe-west6
```


### kubectl connection setup

```bash
$ kubectl config view

apiVersion: v1
clusters: []
contexts: []
current-context: ""
kind: Config
preferences: {}
users: []
```

The used configuration is stored in the `/.kube/config` file. 

Get cluster credentials:

```bash
$ gcloud container clusters get-credentials europe-west6-a-cluster --zone europe-west6-a --project study-pv-subsystem

Fetching cluster endpoint and auth data.
kubeconfig entry generated for europe-west6-a-cluster.
``` 

Let's check our kube config again:

```bash
$ kubectl config view
```

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://34.65.75.145
  name: gke_study-pv-subsystem_europe-west6-c_my-cluster
contexts:
- context:
    cluster: gke_study-pv-subsystem_europe-west6-c_my-cluster
    user: gke_study-pv-subsystem_europe-west6-c_my-cluster
  name: gke_study-pv-subsystem_europe-west6-c_my-cluster
current-context: gke_study-pv-subsystem_europe-west6-c_my-cluster
kind: Config
preferences: {}
users:
- name: gke_study-pv-subsystem_europe-west6-c_my-cluster
  user:
    auth-provider:
      config:
        access-token: ya29.ImapB24hbak5H81pItFkEHBH7J8GC2hxyI5lds8kf129Wsy6pwheM8jAHwi4AxBqP-xiQi9XIbFy4tsytTm6QIMhsCicsG0CFl22jR9lZrQEfv2z86kYLHLzz64KNiM2rLcZSJxIzA0
        cmd-args: config config-helper --format=json
        cmd-path: /usr/local/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/bin/gcloud
        expiry: "2019-10-27T20:50:22Z"
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
```

Get cluster info

```bash
$ kubectl cluster-info
Kubernetes master is running at https://34.65.75.145
GLBCDefaultBackend is running at https://34.65.75.145/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
KubeDNS is running at https://34.65.75.145/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://34.65.75.145/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

Get the cluster nodes

```bash
$ kubectl get nodes
NAME                                  STATUS   ROLES    AGE   VERSION
gke-my-cluster-pool-1-55c9a567-3k1w   Ready    <none>   29m   v1.14.7-gke.10
```

### Storage Class

A PVC dynamically creates a PV using the cluster's default storage class.

```bash
$ kubectl get sc

NAME                 PROVISIONER            AGE
standard (default)   kubernetes.io/gce-pd   46s
```

```bash
$ kubectl describe sc/standard

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

### SCI plugin

Google Persistent Disk (standard or ssd) using the GCEPersistentDisk we can manage Persistent Volume objects.
 
  

### create PersistentDisk 

Resource from: https://cloud.google.com/compute/docs/disks/add-persistent-disk#formatting

```bash
$ gcloud compute disks list

NAME                                                 LOCATION        LOCATION_SCOPE  SIZE_GB  TYPE         STATUS
gke-europe-west6-a-clust-default-pool-e9c84ada-6rl7  europe-west6-a  zone            100      pd-standard  READY
gke-europe-west6-a-clust-default-pool-e9c84ada-90m9  europe-west6-a  zone            100      pd-standard  READY
gke-europe-west6-a-clust-default-pool-e9c84ada-rgsz  europe-west6-a  zone            100      pd-standard  READY
```

```bash
$ gcloud compute disks create wordpress-1 \
    --size 100 \
    --type pd-ssd \
    --zone europe-west6-a
```

We must format and mount the disk before we can use it.

```bash
$ gcloud compute instances attach-disk gke-europe-west6-a-clust-default-pool-e9c84ada-6rl7 --disk wordpress-1
```


SSH into one node

```bash
$ gcloud compute ssh gke-europe-west6-a-clust-default-pool-e9c84ada-6rl7
```

```bash
$ sudo lsblk

NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
...
sdb       8:16   0  100G  0 disk
```

### Format the disk. 

It is recommend a single ext4 file system without a partition table.

```bash
$ sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb

mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done
Creating filesystem with 26214400 4k blocks and 6553600 inodes
Filesystem UUID: ef640f7e-e6f5-4b00-a02e-7d4406f85cc4
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: done
Writing inode tables: done
Creating journal (131072 blocks): done
Writing superblocks and filesystem accounting information: done
```

Create the mount directory 

```bash
$ sudo mkdir -p /mnt/disks/disk0
```


```bash
$ sudo mount -o discard,defaults /dev/sdb /mnt/disks/disk0
```

Configure read and write permissions

```bash
$ sudo chmod a+w /mnt/disks/disk0
```


### Create a PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ssd    
  capacity:
    storage: 20Gi
  gcePersistentDisk:
    pdName: wordpress-1
```

```bash
$ kubectl apply -f gke-pv.yaml

persistentvolume/pv1 created
```

```bash
$ kubectl get pv

NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv1    20Gi       RWO            Retain           Available           ssd                     41s
```

### Create a PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ssd    
  resources:
    requests:
      storage: 20Gi    
```

```bash
$ kubectl apply -f gke-pvc.yaml

persistentvolume/pvc1 created
```

```bash
$ kubectl get pvc

NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc1   Bound    pv1      20Gi       RWO            ssd            23s
```


### Create a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc1
  containers:
    - name: ubuntu-ctr
      image: gcr.io/gcp-runtimes/ubuntu_16_0_4:latest
      command:
        - /bin/bash
        - "-c"
        - "sleep 60m"
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/data"
          name: data
```

```bash
$ kubectl apply -f gke-pod.yaml

pod "pod1" deleted
```

```bash
$ kubectl get pods
```

If it gets stucked in `ContainerCreating` status you can find more information using:

```bash
$ kubectl describe pods
```

Note that you can't attach PersistentDisks in write mode on multiple nodes at the same time.

https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/readonlymany-disks

Needed to change the `PersistentVolumeClaim` to `ReadOnlyMany` accessMode

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 20Gi    
```

The when creating the pod the `PersistentDisk` was automatically mounted to the node

```bash
kubectl get pod -o=custom-columns=NODE:.spec.nodeName
```

```bash

gcloud compute ssh gke-europe-west6-a-clust-default-pool-e9c84ada-90m9
zoal@gke-europe-west6-a-clust-default-pool-e9c84ada-90m9 ~ $ sudo lsblk

NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
...
sdb       8:16   0   20G  0 disk /home/kubernetes/containerized_mounter/rootfs/var/lib/kubelet/pods/a52c1d14-f4ec-11e9-b91f-42010aac0fdc/volumes/kubernetes.io~gce-pd/pvc-8738b0f2-f4ec-11e9-b91
...
```




Deployments vs. StatefulSets

https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes#deployments_vs_statefulsets