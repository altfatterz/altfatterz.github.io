---
layout: post
title: Getting started with kustomize
tags: [kustomize, kubectl, kubernetes]
---

[Kustomize](https://kustomize.io/) is a command line tool that lets you customize application configuration in a template free way.
In Kubernetes world the declarative way is the recommended approach to create the resources. However, it is difficult to use only `kubectl` 
to follow the declarative way, another tools are required like, like [Helm](https://helm.sh/), [Kapitan](https://github.com/deepmind/kapitan), [ktmpl](https://github.com/jimmycuadra/ktmpl).
The full list of these tools you can find [here](https://docs.google.com/spreadsheets/d/1FCgqz1Ci7_VCz_wdh8vBitZ3giBtac_H8SBw4uxnrsE/edit#gid=0)

The drawback of these tools is that you have to learn new complicated DSLs, they use templating which can only override parameterized config, they provide multiple features 
like package or dependency management, come with dashboards which live in your cluster, they allow to manage the lifecycle of specific version when to rollback specific version, and come with customization features too, when you deploy to different environments   

`Kustomize` instead is focusing only on the `customization` domain and allows you to tailor your YAML files to a specific environment.
 It is using `overlay` approach and exposes and teaches the native k8s APIs, not trying to hide them. This makes sure, that the user gets a deeper understanding about Kubernetes.
    
`Kustomize` is available as standalone executable (`kustomize`) and since 1.14 is part of `kubectl` using the `kubectl apply` with `-k` flag.
In this blog post we will use the standalone executable, which can be easily installed following the [guide](https://kubernetes-sigs.github.io/kustomize/installation/) specific to your platform.

For Mac users is straightforward:

```bash
$ brew install kustomize
```

In this article we assume you have already a `Docker` environment and `kubectl` is already installed. To create a Kubernetes cluster we use [kind](https://kind.sigs.k8s.io/docs/user/quick-start/):

```bash
$ kind create cluster --config=kind-cluster-config.yaml
```

where the `kind-cluster-config.yaml` specifies a 3 node cluster.

```yaml
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: my-cluster
nodes:
- role: control-plane
- role: worker
- role: worker
```

The command also creates a `Kubernetes` `context` and switches to it. `Kind` is a great tool to run multiple-node Kubernetes clusters locally. With `kind get clusters` we can query the created clusters.

With `docker ps` you can view the Kubernetes nodes 

```
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                       NAMES
067290147e33        kindest/node:v1.19.1   "/usr/local/bin/entr…"   4 days ago          Up 31 minutes                                   my-cluster-worker
c1ab152ccf6e        kindest/node:v1.19.1   "/usr/local/bin/entr…"   4 days ago          Up 31 minutes       127.0.0.1:61807->6443/tcp   my-cluster-control-plane
9d64cefcd1e7        kindest/node:v1.19.1   "/usr/local/bin/entr…"   4 days ago          Up 31 minutes                                   my-cluster-worker2 
```

Next, we are going to use the [kustomize-demo](https://github.com/altfatterz/kustomize-demo) application which is very simple Spring Boot application in order to showcase `kustomize` features.
After cloning the repository and building the example we have a docker image `kustomize-demo:0.0.1-SNAPSHOT`

```
$ git clone https://github.com/altfatterz/kustomize-demo
$ cd kustomize-demo
$ mvn clean package
```
 
Next we need to load the created Docker image into our Kubernetes cluster.

```bash
$ kind load docker-image kustomize-demo:0.0.1-SNAPSHOT --name my-cluster
altfatterz@zoltan-altfatter-zrh:~/projects/demos/kustomize-demo|master⚡ ⇒  kind load docker-image kustomize-demo:0.0.1-SNAPSHOT --name my-cluster
Image: "kustomize-demo:0.0.1-SNAPSHOT" with ID "sha256:adb9bc431439dea21dc31155587da79183bda9f8636d80127d08d0dbdce6d4c7" not yet present on node "my-cluster-worker", loading...
Image: "kustomize-demo:0.0.1-SNAPSHOT" with ID "sha256:adb9bc431439dea21dc31155587da79183bda9f8636d80127d08d0dbdce6d4c7" not yet present on node "my-cluster-control-plane", loading...
Image: "kustomize-demo:0.0.1-SNAPSHOT" with ID "sha256:adb9bc431439dea21dc31155587da79183bda9f8636d80127d08d0dbdce6d4c7" not yet present on node "my-cluster-worker2", loading...
``` 

We can verify that the image is present on Kubernetes nodes using  

```
$ docker exec -it 067290147e33 crictl images | grep kustomize-demo
docker.io/library/kustomize-demo           0.0.1-SNAPSHOT       adb9bc431439d       259MB  
```

Finally, we create two namespaces where we will deploy the application with different configurations:

```bash
$ kubectl create namespace dev
$ kubectl create namespace prod
```

In the `kustomize-demo/ops` folder we can find the traditional way of declarative Kubernetes configuration duplicating the resource definitions for `dev` and `prod` environments.        

`Kustomize` allows to specify the resource definitions without duplicating common elements. It does this Kubernetes way, using to use Custom Resource Definitions (CRDs) to configure the differences, rather than variable-replacement.     
We move the common yaml configuration into a `base` directory and create two `overlays` representing `dev` and `prod` environments using the following directory structure:     
     
```bash
├── base
│   ├── deployment.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │    └── kustomization.yaml
    └── prod
        └── kustomization.yaml
```

where the `dev/kustomization.yaml` content is 

```bash
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base/deployment.yaml
- ../../base/service.yaml
namespace: dev
```

In order to see the yaml generated for the `dev` environment we can use:

```bash
$ kustomize build --load_restrictor none k8s/overlays/dev
```

The `--load_restrictor` flag set to `none`, allows that customizations may load files from outside their root.

To create the resource in the dev namespace then we can pipe the output of the `kustomize` command to the `kubectl apply` 

```bash
$ kustomize build --load_restrictor none k8s/overlays/dev | kubectl apply -f -
service/kustomize-demo-service created
deployment.apps/kustomize-demo-deployment created
```

To delete the resources we can use:

```bash
$ kustomize build --load_restrictor none k8s/overlays/dev | kubectl delete -f -
```

In production environment is good approach to add a prefix like `prod-` to all product resource names in order to avoid 
modifying or deleting these resources by mistake. With `Kustomize` we can easily achieve this adding the following snippet 
to `prod/kustomization.yaml` configuration file

```yaml
namePrefix: prod- 
```

We want also that resources in production environment to have certain labels so that we can query them by label selector:

```yaml
commonLabels:
  env: prod
```

We also want to increase the replicas count in production environment (`prod/increase_replicas_patch.yaml`) 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kustomize-demo
spec:
  replicas: 3  
```

Another important configuration in production is to set the resource constraints (`prod/resource_constraints_patch.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kustomize-demo
spec:
  template:
    spec:
      containers:
        - name: kustomize-demo
          resources:
            requests:
              memory: 512Mi
              cpu: 256m
            limits:
              memory: 1Gi
              cpu: 512m
```

and health check configuration (`prod/health_check_patch.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kustomize-demo
spec:
  template:
    spec:
      containers:
        - name: kustomize-demo
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

We need to reference all these patches in `prod/kustomization.yaml` under the `patchesStrategicMerge` 

```yaml
patchesStrategicMerge:
- resource_constraints_patch.yaml
- environment_patch.yaml
- health_check_patch.yaml
- increase_replicas_patch.yaml
```

To create the resources in `prod` environment we use

```bash
$ kustomize build --load_restrictor none k8s/overlays/prod | kubectl apply -f -
```

Indeed, the pods where created

```bash
$ kubectl get pods -n prod
NAME                                   READY   STATUS    RESTARTS   AGE
prod-kustomize-demo-548ccc8db9-dtc52   1/1     Running   0          25s
prod-kustomize-demo-548ccc8db9-m9xzb   1/1     Running   0          25s
prod-kustomize-demo-548ccc8db9-qjhxw   1/1     Running   0          25s
```

The source code for this blog post can be found here [https://github.com/altfatterz/kustomize-demo](https://github.com/altfatterz/kustomize-demo) 


