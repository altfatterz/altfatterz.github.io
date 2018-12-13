---
layout: post
title: Working with Kubernetes
tags: [kubernetes, docker]
---

### Install the necessary tools

```bash
$ brew cask install virtualbox
$ brew cask install minikube
$ brew cask install minishift
```

```bash
$ brew install kubernetes-cli
$ brew install openshift-cli
```

### Minikube

Minikube default vm-driver is set to `virtualbox`. You can also specify the kubernetes version
 
```bash
$ minikube start --kubernetes-version v1.13.0
```

You can also set the kubernetes version via

```bash
$ minikube config set kubernetes-version v1.13.0
``` 

Minikube configuration directory is in `~/.minikube` 

Minikube will also create a “minikube” context, and set it to default in kubectl.

```bash
$ kubectl config current-context
minikube
```

```bash
$ kubectl version

Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.0", GitCommit:"ddf47ac13c1a9483ea035a79cd7c10005ff21a6d", GitTreeState:"clean", BuildDate:"2018-12-04T07:48:45Z", GoVersion:"go1.11.2", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.0", GitCommit:"ddf47ac13c1a9483ea035a79cd7c10005ff21a6d", GitTreeState:"clean", BuildDate:"2018-12-03T20:56:12Z", GoVersion:"go1.11.2", Compiler:"gc", Platform:"linux/amd64"}
```

```bash
$ minikube dashboard
```


Minishift default vm-driver is set to `xhyve` 

```bash
$ minishift start --vm-driver virtualbox --openshift-version v3.11.0
```

Creates the directory `~/minishift`

Minishift VM will be configured with 4 GB memory, 2 vCPUs and 20 GB disk size.


Minikube deletes the cluster without any warning

```bash
Login to server ...
Creating initial project "myproject" ...
Server Information ...
OpenShift server started.

The server is accessible via web console at:
    https://192.168.99.101:8443/console

You are logged in as:
    User:     developer
    Password: <any value>

To login as administrator:
    oc login -u system:admin
```



```bash
$ oc version

oc v3.11.0+0cbc58b
kubernetes v1.11.0+d4cacc0
features: Basic-Auth

Server https://192.168.99.101:8443
kubernetes v1.11.0+d4cacc0
```


```bash
$ minikube delete

```

```bash
$ minishift delete
You are deleting the Minishift VM: 'minishift'. Do you want to continue [y/N]?: y

Deleting the Minishift VM...
Minishift VM deleted.
```


### Spring Cloud Kubernetes


### Running app on Minikube


```bash
$ eval $(minikube docker-env)
```

```bash
$ mvn clean package dockerfile:build
```

```bash
$ docker images

REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
altfatterz/kubernetes-hello-world         latest              192e402e7c34        46 seconds ago      121MB
k8s.gcr.io/kube-proxy                     v1.13.0             8fa56d18961f        8 days ago          80.2MB
k8s.gcr.io/kube-scheduler                 v1.13.0             9508b7d8008d        8 days ago          79.6MB
k8s.gcr.io/kube-controller-manager        v1.13.0             d82530ead066        8 days ago          146MB
k8s.gcr.io/kube-apiserver                 v1.13.0             f1ff9b7e3d6e        8 days ago          181MB
k8s.gcr.io/coredns                        1.2.6               f59dcacceff4        5 weeks ago         40MB
openjdk                                   8-jdk-alpine        97bc1352afde        6 weeks ago         103MB
k8s.gcr.io/etcd                           3.2.24              3cab8e1b9802        2 months ago        220MB
k8s.gcr.io/kubernetes-dashboard-amd64     v1.10.0             0dab2435c100        3 months ago        122MB
k8s.gcr.io/kube-addon-manager             v8.6                9c16409588eb        9 months ago        78.4MB
k8s.gcr.io/pause                          3.1                 da86e6ba6ca1        11 months ago       742kB
gcr.io/k8s-minikube/storage-provisioner   v1.8.1              4689081edb10        13 months ago       80.8MB
```

Watching the changes: 

```bash
$ watch kubectl get deployment,svc,pod
```  

Create the deployment

```bash
$ kubectl create deployment kubernetes-hello-world --image=altfatterz/kubernetes-hello-world
```

Tail and stream a single pod 

```bash
$ kubectl logs -f kubernetes-hello-world-deployment-8575589f4d-2mlkf
```

Get the logs from multiple pods with selector

```bash
$ kubectl logs -l app=kubernetes-hello-world
```

Unfortunately it is not possible to combine `-f` and `-l` options with `kubectl logs` command. With [`stern`](https://github.com/wercker/stern) is easy:

```bash
$ stern kubernetes-hello-world-deployment
``` 

Create the service

```bash
$ kubectl create -f kubernetes-hello-world-service.yml
```

Access the service

```bash
$ minikube service kubernetes-hello-world
```

This will open the browser with `<NodeIP>:<NodePort>` in our case `http://192.168.99.100:30001/`

Delete service and deployment

```bash
$ kubectl delete service kubernetes-hello-world
$ kubectl delete deployment kubernetes-hello-world-deployment
```

### Running app on Minishift


### Running app on Kubernetes Engine


### Running app on openshift.com 

You can easily setup a `OpenShift Online Starter` free plan with limited resources. After sign up you will receive a welcome email when the provisioned cluster is ready.


Resources:
https://spring.io/guides/gs/spring-boot-docker/
https://spring.io/guides/topicals/spring-boot-docker/
https://www.baeldung.com/spring-boot-minikube
https://cloudowski.com/articles/10-differences-between-openshift-and-kubernetes/
https://github.com/spotify/dockerfile-maven
https://www.baeldung.com/spring-boot-deploy-openshift
https://blog.containership.io/micronetes/
