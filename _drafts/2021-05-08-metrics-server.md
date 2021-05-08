```bash
$ kind create cluster
```

TODO: Check what pods are running, and check what it returns for commands like:

```bash
$ kubectl top pods
$ kubectl top nodes
```


```bash
$ kubectl apply -f https://github.com/altfatterz/metrics-server/blob/master/components-v0.4.4-with-kubelet-insecure-tls.yaml

serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

```bash
$ kubectl get pods --all-namespaces

kube-system          coredns-74ff55c5b-5sclb                      1/1     Running   0          28m
kube-system          coredns-74ff55c5b-7nhl8                      1/1     Running   0          28m
kube-system          etcd-kind-control-plane                      1/1     Running   0          28m
kube-system          kindnet-gss97                                1/1     Running   1          28m
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          28m
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          28m
kube-system          kube-proxy-smxnk                             1/1     Running   0          28m
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          28m
kube-system          metrics-server-6778f49766-ptjf5              1/1     Running   0          4m7s
local-path-storage   local-path-provisioner-78776bfc44-6xhll      1/1     Running   0          28m
```

Verify that the server is up and running

```bash
$ kubectl get --raw /apis/metrics.k8s.io

{"kind":"APIGroup","apiVersion":"v1","name":"metrics.k8s.io","versions":[{"groupVersion":"metrics.k8s.io/v1beta1","version":"v1beta1"}],"preferredVersion":{"groupVersion":"metrics.k8s.io/v1beta1","version":"v1beta1"}}
```

```bash
$ kubectl top nodes

NAME                 CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
kind-control-plane   281m         7%     598Mi           7%
```

It might takes some time to see data:

```bash
$ kubectl top pods
error: metrics not available yet
```

```bash
$ kubectl top pods

NAME                      CPU(cores)   MEMORY(bytes)
resource-consumer-big     301m         6Mi
resource-consumer-small   100m         6Mi
```

Get data from different namespace

```bash
$ kubectl top pods -n kube-system


coredns-74ff55c5b-5sclb                      4m           8Mi
coredns-74ff55c5b-7nhl8                      5m           8Mi
etcd-kind-control-plane                      20m          32Mi
kindnet-gss97                                1m           5Mi
kube-apiserver-kind-control-plane            87m          249Mi
kube-controller-manager-kind-control-plane   31m          41Mi
kube-proxy-smxnk                             1m           10Mi
kube-scheduler-kind-control-plane            4m           15Mi
metrics-server-6778f49766-ptjf5              5m           12Mi
```

References: 

Tools for Monitoring Resources 
* https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/
