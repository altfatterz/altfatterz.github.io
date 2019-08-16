---
layout: post
title: Deploy to local and remote Kubernetes cluster
tags: [minikube, kubernetes, docker, springboot]
---

In this blog post we look into how to setup the development environment when deploying to a Kubernetes cluster. We will use a simple Spring Boot service which we deploy to a local Kubernetes cluster (using [`Minikube`](https://kubernetes.io/docs/setup/minikube/)) and also to a remote Kubernetes cluster (using [`GKE`](https://cloud.google.com/kubernetes-engine/)).

### Local Kubernetes cluster

First we need to install *Docker for Mac*. See the details [here](https://docs.docker.com/docker-for-mac/)

`Minikube` is the tool which we use to setup a single node Kubernetes cluster running in a VM. We are using `VirtualBox` as the VM driver for `Minikube`   

```bash
$ brew cask install virtualbox
$ brew cask install minikube

```

We will also need `kubectl`, the Kubernetes command-line tool to deploy the service to the local and to a remote Kubernetes cluster.

```bash
$ brew install kubernetes-cli
```

Next, we start the local Kubernetes cluster using the above command. We can also specify the Kubernetes version   
 
```bash
$ minikube start --kubernetes-version v1.13.0
```

or you can specify in the `~/.minikube/config/config.json`

```json
{
    "WantReportError": true,
    "kubernetes-version": "v1.13.0",
    "profile": "minikube"
}
```

Minikube will create a `minikube` context, and set it to default in the `kubeconfig` (~/.kube/config)

```bash
$ kubectl config current-context
minikube
```

There is a handy command to query the version of the client and server versions:

```bash
$ kubectl version

Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.0", GitCommit:"ddf47ac13c1a9483ea035a79cd7c10005ff21a6d", GitTreeState:"clean", BuildDate:"2018-12-04T07:48:45Z", GoVersion:"go1.11.2", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.0", GitCommit:"ddf47ac13c1a9483ea035a79cd7c10005ff21a6d", GitTreeState:"clean", BuildDate:"2018-12-03T20:56:12Z", GoVersion:"go1.11.2", Compiler:"gc", Platform:"linux/amd64"}
```

`Minikube` comes with a `Dasboard` which is a UI for our local Kubernetes cluster and can be accessed using

```bash
$ minikube dashboard
```

Next we create the `Dockerfile` for our demo Spring Boot service. For more information about Spring Boot in a container see Dave Syer's (@david_syer) excellent blog post [https://spring.io/blog/2018/11/08/spring-boot-in-a-container](https://spring.io/blog/2018/11/08/spring-boot-in-a-container) 

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ARG DEPENDENCY=target/dependency
COPY ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY ${DEPENDENCY}/META-INF /app/META-INF
COPY ${DEPENDENCY}/BOOT-INF/classes /app
ENTRYPOINT ["java","-cp","app:app/lib/*","com.example.KubernetesDemoApplication"]
```

and using the `dockerfile-maven-plugin` we build the image:

```bash
$ mvn clean package dockerfile:build
``` 

We are verifying that the docker images is created using

```bash
$ docker images

REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
altfatterz/kubernetes-demo                latest              38fbde4db6b6        2 minutes ago       121MB
openjdk                                   8-jdk-alpine        04060a9dfc39        16 hours ago        103MB
```

and we make sure we can start a container with the image

```bash
$ docker container run -d -p 8080:8080 --name kubernetes-demo altfatterz/kubernetes-demo
45b71ef640967b978614923eec2d017553f134e1f442937a67a188783640d671

$ http :8080
HTTP/1.1 200
Content-Length: 23
Content-Type: text/plain;charset=UTF-8
Date: Thu, 03 Jan 2019 10:33:59 GMT

Happy New Year 2019! :)
```

Let's stop the running docker container using:

```bash
$ docker stop kuberentes-demo
```
  
The minikube VM needs to access this docker image. In order to do this we configure our docker client to point to the minikube docker deamon 

```bash
$ eval $(minikube docker-env)
```

and create the docker image again:

```bash
$ mvn clean package dockerfile:build
```

We can verify that the docker image was created in the `Minikube` environment using:

```bash
$ docker images

REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
altfatterz/kubernetes-demo                latest              38fbde4db6b6        2 minutes ago       121MB
openjdk                                   8-jdk-alpine        04060a9dfc39        16 hours ago        103MB
k8s.gcr.io/kube-proxy                     v1.13.0             8fa56d18961f        2 weeks ago         80.2MB
k8s.gcr.io/kube-scheduler                 v1.13.0             9508b7d8008d        2 weeks ago         79.6MB
k8s.gcr.io/kube-controller-manager        v1.13.0             d82530ead066        2 weeks ago         146MB
k8s.gcr.io/kube-apiserver                 v1.13.0             f1ff9b7e3d6e        2 weeks ago         181MB
k8s.gcr.io/coredns                        1.2.6               f59dcacceff4        6 weeks ago         40MB
k8s.gcr.io/etcd                           3.2.24              3cab8e1b9802        3 months ago        220MB
k8s.gcr.io/kubernetes-dashboard-amd64     v1.10.0             0dab2435c100        3 months ago        122MB
k8s.gcr.io/kube-addon-manager             v8.6                9c16409588eb        10 months ago       78.4MB
k8s.gcr.io/pause                          3.1                 da86e6ba6ca1        12 months ago       742kB
gcr.io/k8s-minikube/storage-provisioner   v1.8.1              4689081edb10        13 months ago       80.8MB
```

For creating the `deployment` and `service` we use the following configuration (`minikube.yml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-demo-deployment
  labels:
    app: kubernetes-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubernetes-demo
  template:
    metadata:
      labels:
        app: kubernetes-demo
    spec:
      containers:
      - name: kubernetes-demo
        image: altfatterz/kubernetes-demo:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: kubernetes-demo
spec:
  type: NodePort
  selector:
    app: kubernetes-demo
  ports:
  - protocol: TCP
    port: 8080
    nodePort: 30001
```

To create these Kubernetes resources on the Kubernetes cluster provided by minikube we use:

```bash
$ kubectl create -f minikube.yml
```

The above configuration creates 3 pods. In order to tail the logs from a single pod we can use the following command: 

```bash
$ kubectl logs -f <pod_name>
```

In order to get the logs from multiple pods we can use the `-l` option with a selector

```bash
$ kubectl logs -l app=kubernetes-demo
```

Unfortunately it is not possible to combine `-f` and `-l` options with `kubectl logs` command. With [`stern`](https://github.com/wercker/stern) is easy:

```bash
$ brew install stern
$ stern kubernetes-demo-deployment
``` 

Another handy utility is the `watch` command to see updates to particular `pod`s, `deployment`s and `service`s 

```bash
$ watch kubectl get deploy,svc,po
```  

We can access the service using:

```bash
$ minikube service kubernetes-demo
```

This will open the browser with `<NodeIP>:<NodePort>` in our case `http://192.168.99.100:30001/`

In order to delete the `service` and `deployment` we use:

```bash
$ kubectl delete service kubernetes-demo
$ kubectl delete deployment kubernetes-demo-deployment
```

### Running the demo service on Kubernetes Engine

Let's install first [Google Cloud SDK](https://cloud.google.com/sdk/)

```bash
$ brew cask install google-cloud-sdk
```

This will install `glcoud` and `gsutil` among other utilities. The `gcloud` is the command-line interface for [GCP](https://cloud.google.com). 
After creating a project and enabling billing at `https://console.cloud.google.com/` initialize `gcloud` setting up the region and zone. 

```bash
$ gcloud init
```

The list of regions and zones you can find here: [https://cloud.google.com/compute/docs/regions-zones/](https://cloud.google.com/compute/docs/regions-zones/). Choose a region close to you, in our case was:

```bash
$ gcloud config list

[compute]
region = europe-west3
zone = europe-west3-a
[core]
account = altfatterz@gmail.com
disable_usage_reporting = True
project = default-project-id

Your active configuration is: [default]
```

Next, configure docker to use the `gcloud` command-line tool as a credential helper

```bash
$ gcloud auth configure-docker
```

The following will be added to your `~/.docker/config.json`

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

The hostname specifies the region of the registry's storage. We are going to use the `eu.gcr.io` to host the images in the EU.

We need to tag the image with the registry name like `[HOSTNAME]/[PROJECT-ID]/[IMAGE]`

```bash
$ docker tag altfatterz/kubernetes-demo eu.gcr.io/default-project-id/kubernetes-demo
```

Next we push the image to [Google's Container Registry](https://cloud.google.com/container-registry/). Here we push the image with the `latest` tag. 

```bash
$ docker push eu.gcr.io/default-project-id/kubernetes-demo
```

List the created image using: 

```bash
$ gcloud container images list --repository=eu.gcr.io/default-project-id

eu.gcr.io/default-project-id/kubernetes-demo
```

You need to use the `--repository` since by default images only from `gcr.io/default-project-id` repository are listed.

Notice, that a bucket in EU location was created: 

```bash
$ gsutil ls

gs://eu.artifacts.default-project-id.appspot.com/
```

We are ready to create a Kubernetes cluster called `demo` using

```bash
$ gcloud container clusters create demo
```

This will create a 3 node cluster of type `n1-standard-1 (1 vCPU, 3.75 GB memory)` running currently by default `1.10.9-gke.5` Kubernetes version using `Container-Optimized OS` image.

After couple of minutes you can get cluster details:

```bash
$ gcloud container clusters list

NAME  LOCATION        MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
demo  europe-west3-a  1.10.9-gke.5    35.246.211.157  n1-standard-1  1.10.9-gke.5  3          RUNNING
```

Behind the scenes it will create `Google Compute Engine` instances and configure each instance as a Kubernetes node.

```bash
$ gcloud compute instances list

NAME                                 ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
gke-demo-default-pool-6067d6ae-20w8  europe-west3-a  n1-standard-1               10.156.0.4   35.246.129.94   RUNNING
gke-demo-default-pool-6067d6ae-8zv7  europe-west3-a  n1-standard-1               10.156.0.3   35.246.135.149  RUNNING
gke-demo-default-pool-6067d6ae-wgmp  europe-west3-a  n1-standard-1               10.156.0.2   35.234.121.25   RUNNING

$ kubectl get nodes

NAME                                  STATUS   ROLES    AGE   VERSION
gke-demo-default-pool-6067d6ae-20w8   Ready    <none>   2m    v1.10.9-gke.5
gke-demo-default-pool-6067d6ae-8zv7   Ready    <none>   2m    v1.10.9-gke.5
gke-demo-default-pool-6067d6ae-wgmp   Ready    <none>   2m    v1.10.9-gke.5
```

When we created the cluster using the `gcloud container clusters create` command it also changed the context which is used by `kubectl`

```bash
$ kubectl config current-context

gke_default-project-id_europe-west3-a_demo
```

If you created the cluster using the [Google Cloud Platform Console](https://console.cloud.google.com/) you can fetch the context information using:

```bash
$ gcloud container clusters get-credentials demo

Fetching cluster endpoint and auth data.
kubeconfig entry generated for hello-world.
```

Later we might want to switch back to `minikube` context, which we can achieve using:

```bash
$ kubectl config use-context minikube
``` 

More information about the location and credentials the `kubectl` knows about our cluster can be returned using:

```bash
$ kubectl config view
``` 

And finally in order to discover the started services we can use:

```bash

$ kubectl cluster-info

Kubernetes master is running at https://35.242.207.215
GLBCDefaultBackend is running at https://35.242.207.215/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
Heapster is running at https://35.242.207.215/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://35.242.207.215/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://35.242.207.215/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

Let's deploy now our demo service to GKE using the following `gke.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-demo-deployment
  labels:
    app: kubernetes-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubernetes-demo
  template:
    metadata:
      labels:
        app: kubernetes-demo
    spec:
      containers:
      - name: kubernetes-demo
        image: eu.gcr.io/default-project-id/kubernetes-demo:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: kubernetes-demo
spec:
  type: LoadBalancer
  selector:
    app: kubernetes-demo
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

with the command:

```bash
$ kubectl create -f gke.yml
```

We wait until a load balancer is provisioned and we get an external IP. Note the used `LoadBalancer` value in the `spec.type` for the service, while in case of minikube this was `NodePort`

```bash
$ kubectl get svc

NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
service/kubernetes-demo   LoadBalancer   10.55.243.231   <pending>       80:30625/TCP   39s
```

and we can test it using:

```bash
$ http <external-ip>
HTTP/1.1 200
Content-Length: 23
Content-Type: text/plain;charset=UTF-8
Date: Thu, 03 Jan 2019 11:12:46 GMT

Happy New Year 2019! :)
```

In the beginning, it might be important sometimes to get a shell to a running container:

```bash
$ kubectl exec -it kubernetes-demo-deployment-656488699d-qsmrt -- sh
/ # wget -q -O- http://localhost:8080/
Happy New Year 2019! :)
/ # exit
```

In development we might want to use the `latest` tag for your images and push the changes to the cluster.
It turns out is not that easy, see discussion [here](https://github.com/kubernetes/kubernetes/issues/27081)

First you make the changes on your code, build the docker image, push the docker image to Google's Container Registry then apply this trick

```bash
$ kubectl patch deployment kubernetes-demo-deployment -p \
  "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}"
```

With the `watch kubectl get pods` command we will see that the pods are re-created. 

### Conclusion

Congratulations! If you followed along you deployed a very simple Spring Boot application to a local and remote Kubernetes cluster. 
The example service you can find on my github account [https://github.com/altfatterz/kubernetes-demo](https://github.com/altfatterz/kubernetes-demo)

Don't forget to delete the demo remote Kubernetes cluster:

```bash
gcloud container clusters delete demo
```