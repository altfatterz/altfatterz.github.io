---
layout: post 
title: Kubernetes Network Policies 
tags: [kubernetes]
---

[Network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) allow to secure a
Kubernetes network. They allow you to control what traffic is allowed to come in or out from your pod. The Kubernetes
network model specifies that every pods gets its own IP address, containers within a pod share the IP address and can
freely communicate with each other. Pods can communicate with all other pods in the cluster using pod IP addresses (without NAT). This style of network is sometimes referred to as a "flat network".

This approach simplifies the network and allows new workloads scheduled dynamically in the cluster with no dependency on
the network design. The network policy represents an important evolution of network security, not just because it handles the dynamic nature of modern
microservices, but because it empowers dev and devops engineers to easily define network security themselves, rather
than needing to learn low-level networking details or raise tickets with a separate team responsible for managing
firewalls.

Every Kubernetes `NetworkPolicy` resource is namespaced and has a pod selector that defines the pods the policies
applies to. (in the below example with label `app: account-service`)
Then it has a series of `ingress` or `egress` rules or both, which defines again with selectors other pods
in the cluster.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: account-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: account-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: account-service-client
      ports:
        - protocol: TCP
          port: 80
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: account-service-db
      ports:
        - protocol: TCP
          port: 5432
``` 

Kubernetes itself does not enforce network policies (just stores them) and instead delegates their enforcement to
network plugins.

Kubernetes built in network
support, [kubenet](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#kubenet)
does not implement network policies. It is common to use third party network implementations which plug into Kubernetes
using the [CNI](https://www.cni.dev/) (Container Network Interface) API.

The most popular CNI plugins are: [flannel](https://github.com/coreos/flannel)
, [calico](https://github.com/projectcalico/calico), [weave](https://github.com/weaveworks/weave)
and [canal](https://docs.projectcalico.org/getting-started/kubernetes/flannel/flannel) (technically a combination of
multiple plugins). [Here](https://rancher.com/blog/2019/2019-03-21-comparing-kubernetes-cni-providers-flannel-calico-canal-and-weave/)
you can find a good article which compares them.

In this blog post we are going to use Canal, but first lets create our Kubernetes cluster.

#### Create our Kubernetes cluster

```bash
$ kind create cluster --name network-policy-demo
Creating cluster "network-policy-demo" ...
 ✓ Ensuring node image (kindest/node:v1.19.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-network-policy-demo"
```

Let's check the running pods in all namespaces by default.

```bash
$ kubectl get pods --all-namespaces

NAMESPACE            NAME                                                        READY   STATUS    RESTARTS   AGE
kube-system          coredns-f9fd979d6-dxbs2                                     1/1     Running   0          73s
kube-system          coredns-f9fd979d6-kl77w                                     1/1     Running   0          73s
kube-system          etcd-network-policy-demo-control-plane                      1/1     Running   0          85s
kube-system          kindnet-vwnbz                                               1/1     Running   0          73s
kube-system          kube-apiserver-network-policy-demo-control-plane            1/1     Running   0          85s
kube-system          kube-controller-manager-network-policy-demo-control-plane   1/1     Running   0          85s
kube-system          kube-proxy-ck6tp                                            1/1     Running   0          73s
kube-system          kube-scheduler-network-policy-demo-control-plane            1/1     Running   0          85s
local-path-storage   local-path-provisioner-78776bfc44-vvhjj                     1/1     Running   0          73s
```

#### Install Calico for policy and flannel (aka Canal) for networking.

Next, we install the network policy support.

```bash
$ curl https://docs.projectcalico.org/manifests/canal.yaml -O
$ kubectl apply -f canal.yaml

configmap/canal-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/canal-flannel created
clusterrolebinding.rbac.authorization.k8s.io/canal-calico created
daemonset.apps/canal created
serviceaccount/canal created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
poddisruptionbudget.policy/calico-kube-controllers created
```

Verify that two more pods were created in the `kube-system` namespace

```bash
$ kubectl get pods --all-namespaces
...
kube-system          calico-kube-controllers-744cfdf676-g82hg                    1/1     Running   0          102s
kube-system          canal-rb8d6                                                 2/2     Running   0          102s
```

Next we define two pods `server` and `client`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: server
  labels:
    app: secure-pod
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

This is a simple `nginx` pod running on port 80 with label `app=secure-pod`. We will try to access this pod from a `client` pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: client
spec:
  containers:
    - name: busybox
      image: radial/busyboxplus:curl
      command: [ "/bin/sh", "-c", "while true; do sleep 3600; done" ]
```

We create both pods on the cluster.

```bash
$ kubectl apply -f server.yaml
$ kubectl apply -f client.yaml
$ kubectl get pods -o wide
NAME     READY   STATUS    RESTARTS   AGE   IP           NODE                                NOMINATED NODE   READINESS GATES
client   1/1     Running   0          10m   10.244.0.7   network-policy-demo-control-plane   <none>           <none>
server   1/1     Running   0          10m   10.244.0.6   network-policy-demo-control-plane   <none>           <none>
```

Then from the `client` pod we can see that we can access the `server` pod using the `server` pod's IP address:

```bash
$ kubectl exec -it client -- curl 10.244.0.6

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Next we can create a network policy selecting our `server` pod with label `app=secure-pod`

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: simple-network-policy
spec:
  podSelector:
    matchLabels:
      app: secure-pod
  ingress:
  - from:
    - podSelector:
        matchLabels:
          allow-access: "true"
    ports:
    - protocol: TCP
      port: 80
```

As soon as a network policy applies to a pod that pod is completely locked down, meaning nothing can talk to it and it cannot talk to anything until we provide some rules that whitelist some specific traffic.
The above network policy will only apply to incoming traffic (pods with label `allow-access: true`) and the traffic leaving the pod will be completely open.

After applying our network policy on the cluster we can see that the `client` pod cannot access anymore the `server` pod.

```bash
$ kubectl apply -f simple-network-policy
$ kubectl exec -it client -- curl 10.244.0.6
^C
```

After adding the `allow-access=true` label to the `client` pod it will meet the `matchLabels` whitelist condition in our network policy and the access will be allowed: 

```bash
$ kubectl label pod client allow-access=true
$ kubectl exec -it client -- curl 10.244.0.6
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

Hopefully with this example is easier to understand network policies and you can play with it yourself. The resources in this example can be found in my github [repo](https://github.com/altfatterz/learning-kubernetes/tree/master/network-policy-demo). 
Next to `podSelector` there is also `namespaceSelector` and `ipBlock` which you can use in the `ingress` or `egress` rules. 
More details you can find in the official [documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/). 



