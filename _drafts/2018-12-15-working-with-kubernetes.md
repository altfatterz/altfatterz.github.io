---
layout: post
title: Working with Kubernetes
tags: [kubernetes, docker]
---

### Install the necessary tools

```bash
$ brew cask install virtualbox
$ brew cask install minikube
$ brew cask install google-cloud-sdk

```

```bash
$ brew install kubernetes-cli

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

To which cluster are we connected with `kubectl`

```bash
$ kubectl cluster-info

Kubernetes master is running at https://192.168.99.101:8443
KubeDNS is running at https://192.168.99.101:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
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
$ brew install stern
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

### Running app on Kubernetes Engine

Let's install first Google Cloud SDK

```bash
$ brew cask install google-cloud-sdk
```

This will install `glcoud` and `gsutil` among other utilities. 

The `gcloud` is the command-line interface for GCP. 

Create a project and enable billing at `https://console.cloud.google.com/`

Initialize the gcloud using in order to set the account, the compute region and zone. 


```bash
$ gcloud init
```

The list of regions and zones you can find here: https://cloud.google.com/compute/docs/regions-zones/
Choose a region close to you.

```bash
$ gcloud config list

[compute]
region = europe-west3
zone = europe-west3-a
[core]
account = altfatterz@gmail.com
disable_usage_reporting = True
project = default-project-id
```

Your active configuration is: [default]

Configure docker to use the `gcloud` command-line tool as a credential helper

```bash
$ gcloud auth configure-docker
```

The following will be added to your `~/.dockker/config.json`

```json
{
  "credHelpers": {
    "gcr.io": "gcloud",
    "us.gcr.io": "gcloud",
    "eu.gcr.io": "gcloud",
    "asia.gcr.io": "gcloud",
    "staging-k8s.gcr.io": "gcloud",
    "marketplace.gcr.io": "gcloud"
  }
}
```

The hostname specifies the region of the registry's storage. We are going to use the `eu.gcr.io` to host the images in the European Union.

Next we need to tag the image with the registry name like `[HOSTNAME]/[PROJECT-ID]/[IMAGE]`

```bash
$ docker tag altfatterz/kubernetes-hello-world eu.gcr.io/default-project-id/kubernetes-hello-world
```

Next we push the image. Here we push the image with the `latest` tag. 

```bash
$ docker push eu.gcr.io/default-project-id/kubernetes-hello-world
```

List the created image using 

```bash
$ gcloud container images list --repository=eu.gcr.io/default-project-id

eu.gcr.io/default-project-id/kubernetes-hello-world
```

You need to use the `--repository` since the by default images only from `gcr.io/default-project-id` repository are listed.

Notice, that this created a bucket in EU location:

```bash
$ gsutil ls

gs://eu.artifacts.default-project-id.appspot.com/
```

```bash
$ kubectl config current-context

minikube
```

Create the Kubernetes cluster called `demo` 

```bash
$ gcloud container clusters create demo
```

This will create a 3 type `n1-standard-1 (1 vCPU, 3.75 GB memory)` node cluster running currently by default `1.10.9-gke.5` Kubernetes version using `Container-Optimized OS` image.

After couple of minutes you can get cluster details:

```bash
$ gcloud container clusters list

NAME         LOCATION        MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
hello-world  europe-west3-a  1.10.9-gke.5    35.242.208.114  n1-standard-1  1.10.9-gke.5  3          RUNNING
```

Behind the scenes it will create Google Compute Engine instances, and configure each instance as a Kubernetes node.

```bash
$ gcloud compute instances list

NAME                                        ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
gke-hello-world-default-pool-c7fef346-cdnq  europe-west3-a  n1-standard-1               10.156.0.3   35.246.206.79   RUNNING
gke-hello-world-default-pool-c7fef346-d6m6  europe-west3-a  n1-standard-1               10.156.0.2   35.198.112.35   RUNNING
gke-hello-world-default-pool-c7fef346-rm4z  europe-west3-a  n1-standard-1               10.156.0.4   35.242.253.194  RUNNING
```

We can add the connection details to `kubectl` using:

```bash
$ gcloud container clusters get-credentials hello-world 

Fetching cluster endpoint and auth data.
kubeconfig entry generated for hello-world.
```

This changes the current-context for `kubectl` (previously was `minikube`)

```bash
$ kubectl config current-context

gke_default-project-id_europe-west3-a_hello-world
```

```bash
$ kubectl cluster-info

Kubernetes master is running at https://35.242.208.114
GLBCDefaultBackend is running at https://35.242.208.114/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
Heapster is running at https://35.242.208.114/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://35.242.208.114/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://35.242.208.114/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

```bash
$ kubectl get pod

NAME                                                READY   STATUS    RESTARTS   AGE
kubernetes-hello-world-deployment-86f888b7f-fwgqk   1/1     Running   0          11m
kubernetes-hello-world-deployment-86f888b7f-hhkct   1/1     Running   0          11m
kubernetes-hello-world-deployment-86f888b7f-ktkkp   1/1     Running   0          11m
```

Get a shell to the running container: (the pod contains only one container, no need to use the `-c` option )

```bash
$ kubectl exec -it kubernetes-hello-world-deployment-86f888b7f-fwgqk -- /bin/sh
/ # wget -q -O- http://localhost:8080/
Hello World!
/ # exit
```

Next we expose the service:

```bash
$ kubectl create -f kubernetes-hello-world-gcp-service.yml
```

The service needs to be externally accessible. In Kubernetes, you can instruct the underlying infrastructure to create an external load balancer, by specifying the Service Type as a `LoadBalancer`.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: kubernetes-hello-world
spec:
  type: LoadBalancer
  selector:
    app: kubernetes-hello-world
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

It might take couple of minutes while the external load balancer is created. You should see an ip address in the `EXTERNAL-IP` field as below for the `kubernetes-hello-world` service

```bash
zoal@zoltans-macbook-pro:~|⇒  kubectl get svc

NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
kubernetes               ClusterIP      10.63.240.1     <none>          443/TCP        1h
kubernetes-hello-world   LoadBalancer   10.63.247.113   35.234.125.62   80:30018/TCP   14m```
```

Then you access it like:

```bash
zoal@zoltans-macbook-pro:~|⇒  http http://35.234.125.62

HTTP/1.1 200
Content-Length: 12
Content-Type: text/plain;charset=UTF-8
Date: Fri, 14 Dec 2018 15:25:34 GMT

Hello World!
```

### Update

In development you might want to use the `latest` tag for your images and push the changes to the cluster.
It turns out is not that easy, see discussion here: https://github.com/kubernetes/kubernetes/issues/27081

First you make the changes on your code, build the docker image, push the docker image to Google's Container Registry then apply this trick

```bash
$ kubectl patch deployment kubernetes-hello-world-deployment -p \
  "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}"
```

If you use `watch kubectl get pods` command you will see that the pods are re-created. 

### Conclusion






Resources:
https://spring.io/guides/gs/spring-boot-docker/
https://spring.io/guides/topicals/spring-boot-docker/
https://www.baeldung.com/spring-boot-minikube
https://cloudowski.com/articles/10-differences-between-openshift-and-kubernetes/
https://github.com/spotify/dockerfile-maven
https://www.baeldung.com/spring-boot-deploy-openshift
https://blog.containership.io/micronetes/
http://blog.christianposta.com/microservices/netflix-oss-or-kubernetes-how-about-both/