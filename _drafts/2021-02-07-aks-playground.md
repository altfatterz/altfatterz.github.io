---
layout: post 
title: Scaling options for applications in AKS 
tags: [kubernetes, kas]
---

Horizontal Pod Autoscaler (HPA)
Cluster Autoscaler

Create a resource group:

```bash
$ az group create -n westeurope-rg -l westeurope
```

List resource groups:

```bash
$ az group list -o table
```

Create a container registry

```bash
$ az acr create --resource-group  westeurope-rg --name altfatterz --sku Basic
```

The Basic SKU is a cost-optimized entry point for development purposes that provides a balance of storage and
throughput.

You might get an error message:

The registry DNS name westeuropeacr.azurecr.io is already in use. You can check if the name is already claimed using
following API: https://docs.microsoft.com/en-us/rest/api/containerregistry/registries/checknameavailability

```bash
$ az acr login --name altfatterz
```

To get the `AcrLoginServer` issue the following command:

```bash
$ az acr list --resource-group westeurope-rg --query "[].{acrLoginServer:loginServer}" --output table
```

Push the image

```bash
$ docker push altfatterz.azurecr.io/azure-vote-front:v1
```

```bash
$ az acr repository list --name altfatterz --output table
$ az acr repository show-tags --name altfatterz --repository azure-vote-front --output table
```

Create Kubernetes cluster

```bash
$ az aks create --resource-group westeurope-rg --name westeurope-aks --attach-acr altfatterz 
```

Error:
An RSA key file or key value must be supplied to SSH Key Value. You can use --generate-ssh-keys to let CLI generate one
for you

```bash
$ az aks create --resource-group westeurope-rg --name westeurope-aks --attach-acr altfatterz --generate-ssh-keys 
```

SSH key files '/Users/altfatterz/.ssh/id_rsa' and '/Users/altfatterz/.ssh/id_rsa.pub' have been generated under ~/.ssh
to allow SSH access to the VM. If using machines without permanent storage like Azure Cloud Shell without an attached
file share, back up your keys to a safe location B

Operation could not be completed as it results in exceeding approved Total Regional Cores quota.

```bash
$ az aks create --resource-group westeurope-rg --name westeurope-aks --attach-acr altfatterz --node-count 1 --generate-ssh-keys
```

----

Create Kubernetes cluster (with default kubernetes version, default SKU load balancer (Standard) and default vm set
type, and grant the 'acrpull' role assignment to the ACR specified). Generate SSH public and private key files if
missing. The keys will be stored in the ~/.ssh directory.

```bash
$ az aks create \
--resource-group  westeurope-rg \
--name westeurope-aks \
--node-count 3 \
--node-vm-size Standard_DS1_v2 \
--attach-acr altfatterz \
--generate-ssh-keys \
--enable-cluster-autoscaler \
--min-count 3 \
--max-count 5
```

See DSv2-series here: https://docs.microsoft.com/en-us/azure/virtual-machines/dv2-dsv2-series

```
$ BadRequestError: Operation failed with status: 'Bad Request'. Details: System node pool must use VM sku with more than 2 cores and 4GB memory. Nodepool name: nodepool1.
```





Create an AKS cluster:

```
$ az aks create \
   --resource-group  westeurope-rg \
   --name westeurope-aks \
   --node-count 1 \
   --attach-acr altfatterz \
   --generate-ssh-keys 
```

The above command will create a one node Kubernetes cluster with default kubernetes version, default SKU load balancer (Standard) and default vm set
type (VirtualMachineScaleSets), and grants the 'acrpull' role assignment to the ACR specified. It also generates SSH public and private key files if
missing, the keys will be stored in the ~/.ssh directory. The one node will belong to the system node pool called `nodepool1`.

Let's connect to this cluster to find out more

```bash
$ az aks get-credentials --resource-group westeurope-rg --name westeurope-aks
```

Now we can get more information about our node:

```bash
$ kubectl get nodes -o wide
NAME                                STATUS   ROLES   AGE     VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
aks-nodepool1-28842295-vmss000000   Ready    agent   6m33s   v1.18.14   10.240.0.4    <none>        Ubuntu 18.04.5 LTS   5.4.0-1035-azure   docker://19.3.14

$ kubectl describe node aks-nodepool1-28842295-vmss000000 

Name:               aks-nodepool1-28842295-vmss000000
Roles:              agent
Labels:             agentpool=nodepool1
                    beta.kubernetes.io/instance-type=Standard_DS2_v2
                    kubernetes.azure.com/mode=system
                    ...
Taints:             <none>
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
...
````

We see also that all critical system pods such as `coredns` or `metrics-server` are running on this single node

```bash
$ kubectl  get pods --all-namespaces -o wide

NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE   IP           NODE                                NOMINATED NODE   READINESS GATES
kube-system   coredns-748cdb7bf4-njkk8              1/1     Running   0          10m   10.244.0.5   aks-nodepool1-28842295-vmss000000   <none>           <none>
kube-system   coredns-748cdb7bf4-vgzqt              1/1     Running   0          12m   10.244.0.6   aks-nodepool1-28842295-vmss000000   <none>           <none>
kube-system   coredns-autoscaler-868b684fd4-nhqbp   1/1     Running   0          12m   10.244.0.4   aks-nodepool1-28842295-vmss000000   <none>           <none>
kube-system   kube-proxy-lvdpf                      1/1     Running   0          11m   10.240.0.4   aks-nodepool1-28842295-vmss000000   <none>           <none>
kube-system   metrics-server-58fdc875d5-w7jmf       1/1     Running   0          12m   10.244.0.3   aks-nodepool1-28842295-vmss000000   <none>           <none>
kube-system   tunnelfront-7c4b5dd57d-4r8t8          1/1     Running   0          12m   10.244.0.2   aks-nodepool1-28842295-vmss000000   <none>           <none>
```

In AKS the nodes are grouped into `node pools`. `System node pools` are used for hosting of hosting critical system pods, while `user node pools` serve the primary purpose of hosting application pods.
We saw above that a node from a system node pool has the label `kubernetes.azure.com/mode=system`. However, application pods can be scheduled on system node pools, in order to forbid this we need to taint the node using `CriticalAddonsOnly=true:NoSchedule

```bash
$ az aks nodepool add \
    --resource-group westeurope-rg \
    --cluster-name westeurope-aks \
    --name systempool \
    --node-count 1 \
    --node-taints CriticalAddonsOnly=true:NoSchedule \
    --mode System
```

With the above command we created another system node pool but with a tainted node. Now we have two system node pools:

```bash
$ az aks nodepool list --resource-group westeurope-rg --cluster-name westeurope-aks --output table
Name        OsType    KubernetesVersion    VmSize           Count    MaxPods    ProvisioningState    Mode
----------  --------  -------------------  ---------------  -------  ---------  -------------------  ------
nodepool1   Linux     1.18.14              Standard_DS2_v2  1        110        Succeeded            System
systempool  Linux     1.18.14              Standard_DS2_v2  1        110        Succeeded            System
```

Let's remove the `nodepool1`

```bash
$ az aks nodepool delete --resource-group westeurope-rg --cluster-name westeurope-aks --name nodepool1 
```

If you watch meanwhile the critical system pods in another terminal (kubectl get pods --all-namespaces -o wide --watch), you should see that the pods are terminated and rescheduled on the node from systempool. 

Now let's try to deploy our application to the cluster. Do you think it will work?  

```bash
$ az acr login --name altfatterz
$ docker tag fibo-service:0.0.1-SNAPSHOT altfatterz.azurecr.io/fibo-service:0.0.1-SNAPSHOT
$ docker push altfatterz.azurecr.io/fibo-service:0.0.1-SNAPSHOT 
$ kubectl apply -k k8s/overlay/azure  -- TODO to use kustomize
```

The pod will be in `Pending` state forever since we tainted the node to only run critical system pods only. Delete the resources via `kubectl delete k8s/overlay/azure`  

Now let's add a 3 node `user node pool` to the cluster, where we enable also the cluster autoscaler with the default profile settings. 

```bash
$ az aks nodepool add \
    --resource-group westeurope-rg \
    --cluster-name westeurope-aks \
    --name userpool \
    --node-count 3 \
    --node-vm-size Standard_D1_v2 \
    --enable-cluster-autoscaler \
    --min-count 3 \
    --max-count 5
```

Show the nodepool list:

```bash
$ az aks nodepool list --resource-group westeurope-rg --cluster-name westeurope-aks --output table

Name         OsType    KubernetesVersion    VmSize           Count    MaxPods    ProvisioningState    Mode
-----------  --------  -------------------  ---------------  -------  ---------  -------------------  ------
systempool   Linux     1.18.14              Standard_DS2_v2  1        110        Succeeded            System
userpool     Linux     1.18.14              Standard_D1_v2   3        110        Succeeded            User
```

```bash
$ kubectl get nodes -o wide

NAME                                  STATUS   ROLES   AGE     VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
aks-systempool-28842295-vmss000000    Ready    agent   25m     v1.18.14   10.240.0.5    <none>        Ubuntu 18.04.5 LTS   5.4.0-1035-azure   docker://19.3.14
aks-userpool-28842295-vmss000000      Ready    agent   3m18s   v1.18.14   10.240.0.4    <none>        Ubuntu 18.04.5 LTS   5.4.0-1035-azure   docker://19.3.14
aks-userpool-28842295-vmss000001      Ready    agent   4m13s   v1.18.14   10.240.0.6    <none>        Ubuntu 18.04.5 LTS   5.4.0-1035-azure   docker://19.3.14
aks-userpool-28842295-vmss000002      Ready    agent   3m31s   v1.18.14   10.240.0.7    <none>        Ubuntu 18.04.5 LTS   5.4.0-1035-azure   docker://19.3.14
```

```bash
$ az aks list --output table 
$ az aks nodepool list --resource-group westeurope-rg --cluster-name westeurope-aks
```

```bash
$ az aks nodepool add \
    --resource-group  westeurope-rg \
    --cluster-name westeurope-aks \ 
    --name mynodepool \
    --node-count 3 \
    --node-vm-size Standard_DS1_v2 
```

Update cluster to enable cluster autoscaler

```bash
az aks update \
  --resource-group  westeurope-rg \
  --name westeurope-aks \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 5
```

Disable the cluster autoscaler

```bash
az aks update \
  --resource-group westeurope-rg \
  --name westeurope-aks \
  --disable-cluster-autoscaler
```


```bash
$ kubectl get configmap -n kube-system cluster-autoscaler-status -o yaml
```



```bash
$ az aks list --output table

Name            Location    ResourceGroup    KubernetesVersion    ProvisioningState    Fqdn
--------------  ----------  ---------------  -------------------  -------------------  -----------------------------------------------------------------
westeurope-aks  westeurope  westeurope-rg    1.18.14              Succeeded            westeurope-westeurope-rg-7ac273-d403318c.hcp.westeurope.azmk8s.io
```

Get the credentials:

```bash
$ az aks get-credentials --resource-group westeurope-rg --name westeurope-aks
```

```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://westeurope-westeurope-rg-7ac273-d403318c.hcp.westeurope.azmk8s.io:443
CoreDNS is running at https://westeurope-westeurope-rg-7ac273-d403318c.hcp.westeurope.azmk8s.io:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://westeurope-westeurope-rg-7ac273-d403318c.hcp.westeurope.azmk8s.io:443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

```bash
$ kubectl get nodes -o wide
```

Cleanup

```bash
$ az aks delete --name westeurope-aks --resource-group westeurope-rg 
```

Resource:

1. https://www.replex.io/blog/kubernetes-in-production-best-practices-for-cluster-autoscaler-hpa-and-vpa
2. https://www.novatec-gmbh.de/en/blog/scale-your-spring-boot-application-in-kubernetes/
3. https://piotrminkowski.com/2020/11/05/spring-boot-autoscaling-on-kubernetes/
4. https://www.mokkapps.de/blog/monitoring-spring-boot-application-with-micrometer-prometheus-and-grafana-using-custom-metrics/
5. https://stackabuse.com/monitoring-spring-boot-apps-with-micrometer-prometheus-and-grafana/
6. Really good one:
   https://medium.com/faun/java-application-optimization-on-kubernetes-on-the-example-of-a-spring-boot-microservice-cf3737a2219c


7. https://thorsten-hans.com/aks-cluster-auto-scaler-inside-out#what-is-the-aks-cluster-auto-scaler
8. Cluster Autoscaler
   settings: https://docs.microsoft.com/en-us/azure/aks/cluster-autoscaler#using-the-autoscaler-profile
9. https://www.rubrik.com/en/blog/architecture/20/12/customized-autoscaling--minimize-your-cloud-cost