---
layout: post
title: Hazelcast on Kubernetes
tags: [hazelcast, kubernetes]
---

### hazelcast-demo



#### Setup locally


#### Setup on Kubernetes

Create a Kubernetes cluster with `kind`

```bash
$ kind create cluster --name hazelcast-demo-cluster

Creating cluster "hazelcast-demo-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.19.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-hazelcast-demo-cluster"
```

It automatically switched the kubectl context to `kind-hazelcast-demo-cluster`. To check the `cluster-info` use:

```
$ kubectl cluster-info

Kubernetes control plane is running at https://127.0.0.1:50709
KubeDNS is running at https://127.0.0.1:50709/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```




Create a Docker image from our application.

```bash
$ mvn spring-boot:build-image
$ docker images | grep hazelcast-demo
hazelcast-demo             0.0.1-SNAPSHOT    bd06a8345bbb   41 years ago   291MB
```

Load the created Docker image into our Kubernetes cluster

```bash
$ kind load docker-image hazelcast-demo:0.0.1-SNAPSHOT --name hazelcast-demo-cluster
Image: "hazelcast-demo:0.0.1-SNAPSHOT" with ID "sha256:bd06a8345bbb02707b261cf0cc28ab0104f81f6feeefc89a900e99b74a50ac61" not yet present on node "hazelcast-demo-cluster-control-plane", loading...
$ docker exec -it hazelcast-demo-cluster-control-plane crictl images | grep hazelcast-demo
docker.io/library/hazelcast-demo           0.0.1-SNAPSHOT       bd06a8345bbb0       297MB
```

Create the Kubernetes resources:

```bash
$ mkdir k8s
$ kubectl create deployment hazelcast-demo --image hazelcast-demo:0.0.1-SNAPSHOT -o yaml --dry-run=client > k8s/deployment.yaml
$ kubectl create service clusterip hazelcast-demo --tcp 8080:8080 -o yaml --dry-run=client > k8s/service.yaml
```

Apply the Kubernetes resources:

```bash
$ kubectl apply -f ./k8s
```

Granting Permissions to use `Kubernetes API` which is one of two options of how Hazelcast members discover each other on Kubernetes, next to `DNS Lookup`
`Kubernetes API` mode means that each node makes a REST call to Kubernetes Master in order to discover IPs of PODs.
`DNS Lookup` mode uses a feature of Kubernetes that headless (without cluster IP) services are assigned a DNS record which resolves to the set of IPs of related PODs. (`clusterIP: None`)

```bash
$ kubectl apply -f https://raw.githubusercontent.com/hazelcast/hazelcast-kubernetes/master/rbac.yaml
```


Exception if we don't include the `hazelcast-kubernetes` dependency.

```bash
Caused by: com.hazelcast.config.properties.ValidationException: There is no discovery strategy factory to create 'DiscoveryStrategyConfig{properties={service-label-value=dev, service-label-name=hazelcast-cluster-name, service-port=5701}, className='com.hazelcast.kubernetes.HazelcastKubernetesDiscoveryStrategy', discoveryStrategyFactory=null}' Is it a typo in a strategy classname? Perhaps you forgot to include implementation on a classpath?
	at com.hazelcast.spi.discovery.impl.DefaultDiscoveryService.buildDiscoveryStrategy(DefaultDiscoveryService.java:198) ~[hazelcast-4.1.1.jar:4.1.1]
	at com.hazelcast.spi.discovery.impl.DefaultDiscoveryService.loadDiscoveryStrategies(DefaultDiscoveryService.java:141) ~[hazelcast-4.1.1.jar:4.1.1]
	... 45 common frames omitted
```

Warning because Kubernetes API access is forbidden!

2021-01-06 16:37:32.430  WARN 1 --- [           main] c.hazelcast.kubernetes.KubernetesClient  : Kubernetes API access is forbidden! Starting standalone. To use Hazelcast Kubernetes discovery, configure the required RBAC. For 'default' service account in 'default' namespace execute: `kubectl apply -f https://raw.githubusercontent.com/hazelcast/hazelcast-kubernetes/master/rbac.yaml`


More info about Kubernetes Hazelcast Discovery Extension:
https://community.appway.com/screen/kb/article/kubernetes-hazelcast-discovery-extension-1748003143711



Interesting side effect: when work VPN was enabled, the hazelcast nodes did not form a cluster.

Resources:
* Hazelcast on Kubernetes https://hazelcast.com/blog/how-to-set-up-your-own-on-premises-hazelcast-on-kubernetes/
* https://stackoverflow.com/questions/51622308/multiple-hazelcast-instances-on-same-machine
* WARNING: An illegal reflective access operation has occurred #13151
  https://github.com/hazelcast/hazelcast/issues/13151


If you start app the application you will see a warning about [illegal reflective access](https://github.com/hazelcast/hazelcast/issues/13151).

```bash
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by com.hazelcast.internal.networking.nio.SelectorOptimizer (file:/Users/altfatterz/.m2/repository/com/hazelcast/hazelcast/4.1.1/hazelcast-4.1.1.jar) to field sun.nio.ch.SelectorImpl.selectedKeys
WARNING: Please consider reporting this to the maintainers of com.hazelcast.internal.networking.nio.SelectorOptimizer
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
```

In order to fix this warning set this in the `VM options`:

```bash
--add-modules java.se --add-exports java.base/jdk.internal.ref=ALL-UNNAMED --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.nio=ALL-UNNAMED --add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens java.management/sun.management=ALL-UNNAMED --add-opens jdk.management/com.sun.management.internal=ALL-UNNAMED
```

It is useful to turn on `debug` level logs of hazelcast to understand what is going on under the hood:

```bash
logging.level:
  com.hazelcast: debug
```

Let's start up another instance of the application on port 8081 with program argument `--server.port=8081`. In the logs we can see that they joined the `local` hazelcast cluster.

```bash
2021-01-08 14:12:08.543  INFO 58850 --- [ration.thread-0] c.h.internal.cluster.ClusterService      : [192.168.0.11]:5701 [local] [4.1.1]

Members {size:2, ver:2} [
  Member [192.168.0.11]:5701 - 1cb4dea1-d513-49f7-a503-009cc6e18340 this
  Member [192.168.0.11]:5702 - d9c255fd-fe82-4a78-b51b-4ca43f1e37a0
]
```

Java configuration

```bash
@Bean
@Profile("default")
public Config config() {
  return new Config().setClusterName("local");
}
```

You will see that on Kubernetes the nodes don't for a hazelcast cluster.
Hazelcast Kubernetes plugin is using `Kubernetes API` discovery by default, which means that each Hazelcast nodes makes a REST call to Kubernetes Master in order to discover IPs of the pods.
However, this requires granting certain permissions. To grant for 'default' service account in 'default' namespace we need to create the following to resources:

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: hazelcast-cluster-role
rules:
  - apiGroups:
      - ""
    resources:
      - endpoints
      - pods
      - nodes
      - services
    verbs:
      - get
      - list

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: hazelcast-cluster-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: hazelcast-cluster-role
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
```

We can simply execute this using:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/hazelcast/hazelcast-kubernetes/master/rbac.yaml
```

In the logs now we can see that they joined the cluster:

```bash
2021-01-08 15:09:23.084  INFO 1 --- [ration.thread-0] c.h.internal.cluster.ClusterService      : [10.244.0.12]:5701 [dev] [4.1.1]

Members {size:2, ver:2} [
	Member [10.244.0.11]:5701 - 51a20546-e4cf-45ea-b066-be6d75436825
	Member [10.244.0.12]:5701 - fb445158-a829-416b-af5e-caabd891640b this
]
```


Port forward:

```bash
$ kubectl port-forward svc/hazelcast-demo-service 8080:8080
```

```bash
$ time http :8080/books/123
... // TODO include here a nice book name
3.879 total
$ time http :8080/books/123
... // TODO include here a nice book name
0.572 total
$ time http :8080/books/1234
... // TODO include here another nice book name
3.547 total
```



