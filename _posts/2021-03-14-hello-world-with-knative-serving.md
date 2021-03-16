---
layout: post
title: Hello World with Knative Serving
tags: [knative, kubernetes, kubernerds ]
---

[Knative](https://knative.dev/) goal is to make developers more productive by providing higher level abstractions ([CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)) on top of Kubernetes.
It solves common problems like stand up a scalable, secure, stateless service in seconds, connecting disparate systems together.
`Knative` also brings serverless deployment models to Kubernetes.

There are two major subprojects of Knative: `Serving` and `Eventing`. `Serving` is responsible
for deploying containers on Kubernetes, taking care of the details of networking, autoscaling (even to zero), upgrading, routing.
Eventing is responsible for connecting disparate systems. In early blog posts about Knative it was mentioned `Build` as a third subproject. `Build` became and independent project which is known under the name [Tekton](https://tekton.dev/).

In this blog post we will look into `Knative Serving`. The three main components of `Knative Serving` are good represented on this diagram
(taken from [Jacques Chester](https://twitter.com/jacques_chester)'s excellent book [Knative in Action](https://livebook.manning.com/book/knative-in-action/chapter-1/v-6/184)):

<p><img src="/images/2021-03-14/Knative.png" alt="Knative" /></p>

I like also the Knative definition from [Piotr Mińkowski](https://twitter.com/piotr_minkowski) :

<p><img src="/images/2021-03-14/Knative-CloudFoundry.png" alt="CloudRun" /></p>

### Local setup

In this example we use the single-node Kubernetes cluster provided with [Docker Desktop](https://www.docker.com/products/docker-desktop).
We also configured `Docker Desktop` with at least 4 CPUs and 8 GB memory to make sure everything is running smoothly.

We need `kubectl` which can be easily installed following [https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/) 
and we need also the `kn`, the Knative CLI, which can be easily installed following [https://knative.dev/docs/install/install-kn/](https://knative.dev/v0.19-docs/install/install-kn/) 

Next we can start with Knative installation following [https://knative.dev/docs/install/any-kubernetes-cluster/](https://knative.dev/docs/install/any-kubernetes-cluster/)

```bash
$ kubectl apply --filename https://github.com/knative/serving/releases/download/v0.21.0/serving-crds.yaml
$ kubectl apply --filename https://github.com/knative/serving/releases/download/v0.21.0/serving-core.yaml
```

For networking we use Istio:

```bash
$ kubectl apply --filename https://github.com/knative/net-istio/releases/download/v0.21.0/istio.yaml
$ kubectl apply --filename https://github.com/knative/net-istio/releases/download/v0.21.0/net-istio.yaml
```

For DNS magic we configure Knative Serving to use [xip.io](http://xip.io/) as the default DNS suffix.

```bash
$ kubectl apply --filename https://github.com/knative/serving/releases/download/v0.21.0/serving-default-domain.yaml
```
This is a service that reflects back IP addresses that we submitted as domain names. For example

```bash
$ nslookup 10.1.2.3.xip.io
Server:		192.168.0.254
Address:	192.168.0.254#53

Non-authoritative answer:
Name:	10.1.2.3.xip.io
Address: 10.1.2.3
```
We can use this to send traffic to endpoints for which we haven’t configured a domain name.

After successful installation we have something like this:

```bash
$ kubectl get pods --all-namespaces
NAMESPACE         NAME                                     READY   STATUS      RESTARTS   AGE
istio-system      istio-ingressgateway-7f6b78d5b7-2fzcg    1/1     Running     0          10m
istio-system      istiod-7fcb569c74-8bgs8                  1/1     Running     0          10m
istio-system      istiod-7fcb569c74-mkhcm                  1/1     Running     0          9m38s
istio-system      istiod-7fcb569c74-t9mb9                  1/1     Running     0          9m38s
knative-serving   activator-86956bbd6f-8vzz8               1/1     Running     0          10m
knative-serving   autoscaler-54cbd576f6-rnvpd              1/1     Running     0          10m
knative-serving   controller-79c9cccd6f-648nn              1/1     Running     0          10m
knative-serving   default-domain-rfrfd                     0/1     Completed   0          9m58s
knative-serving   istio-webhook-56748b47-wlp7g             1/1     Running     0          10m
knative-serving   networking-istio-5db557d5c4-xmj86        1/1     Running     0          10m
knative-serving   webhook-5fd484cf4-qgcll                  1/1     Running     0          10m
kube-system       coredns-f9fd979d6-2vd95                  1/1     Running     3          4d16h
kube-system       coredns-f9fd979d6-jjbsm                  1/1     Running     3          4d16h
kube-system       etcd-docker-desktop                      1/1     Running     3          4d16h
kube-system       kube-apiserver-docker-desktop            1/1     Running     3          4d16h
kube-system       kube-controller-manager-docker-desktop   1/1     Running     3          4d16h
kube-system       kube-proxy-cpccf                         1/1     Running     3          4d16h
kube-system       kube-scheduler-docker-desktop            1/1     Running     3          4d16h
kube-system       storage-provisioner                      1/1     Running     5          4d16h
kube-system       vpnkit-controller                        1/1     Running     3          4d16h
```

### Example application

We use a [simple](https://github.com/altfatterz/cloud-run-demos/tree/master/hello-world-knative) Spring Boot application using the [recently released](https://spring.io/blog/2021/03/11/announcing-spring-native-beta) `Spring Native Beta`.
After building the image (note it will take longer, around 5 minutes)

```bash
$ mvn spring-boot:build-image
```

we can start our application using

```bash
$ docker run --rm -p 8080:8080 altfatterz/hello-world-knative:0.0.1-SNAPSHOT
... 
2021-03-15 20:05:47.331  INFO 1 --- [           main] c.example.HelloWorldKnativeApplication   : Started HelloWorldKnativeApplication in 0.054 seconds (JVM running for 0.057)
```

Note, the application starts up super fast only 0.054 seconds.

Next, we need to push our image into a Docker Registry. We use [Dockerhub](https://hub.docker.com/) in this example.

```bash
$ docker login
$ docker push altfatterz/hello-world-knative:0.0.1-SNAPSHOT
```

### Knative Serving

We create a Knative Service using:

```bash
$ kn service create hello-world-knative --image=docker.io/altfatterz/hello-world-knative:0.0.1-SNAPSHOT --env TARGET="Knative"
```

```bash
Creating service 'hello-world-knative' in namespace 'default':

  0.049s The Route is still working to reflect the latest desired specification.
  0.085s ...
  0.094s Configuration "hello-world-knative" is waiting for a Revision to become ready.
 15.504s ...
 15.561s Ingress has not yet been reconciled.
 15.645s Waiting for load balancer to be ready
 15.835s Ready to serve.

Service 'hello-world-knative' created to latest revision 'hello-world-knative-00001' is available at URL:
http://hello-world-knative.default.127.0.0.1.xip.io
```

`kn` creates a `Knative Service`, which causes the creation of a `Route`, a `Revision` and a `Configuration`

```bash
$ kn service list
NAME                  URL                                                   LATEST                      AGE   CONDITIONS   READY   REASON
hello-world-knative   http://hello-world-knative.default.127.0.0.1.xip.io   hello-world-knative-00001   45m   3 OK / 3     True

$ kn routes list
NAME                  URL                                                   READY
hello-world-knative   http://hello-world-knative.default.127.0.0.1.xip.io   True

$ kn revisions list
NAME                        SERVICE               TRAFFIC   TAGS   GENERATION   AGE     CONDITIONS   READY   REASON
hello-world-knative-00001   hello-world-knative   100%             1            2m19s   3 OK / 4     True
```

The `3 OK / 4 ` value in the the `Route` conditions column could be more understood if we ask for the details of the revision

```bash
$ kn revision hello-world-knative-00001

Name:       hello-world-knative-00001
Namespace:  default
Age:        51m
Image:      docker.io/altfatterz/hello-world-knative:0.0.1-SNAPSHOT (pinned to 913d61)
Env:        TARGET=Knative
Service:    hello-world-knative

Conditions:
  OK TYPE                  AGE REASON
  ++ Ready                 51m
  ++ ContainerHealthy      51m
  ++ ResourcesAvailable    51m
   I Active                 3m NoTraffic  
```

In the `Conditions` the `OK` column `++` means everything is ok, `!!` means that something is bad, `??` when Knative has no clue what is happening.
The `Active` condition is very important, it tells us if an instance of the `Revision` is running. In this case not, indeed if we use:

```bash
$ kubectl get pods
No resources found in default namespace.
```

As soon as, we send an HTTP request via `curl http://hello-world-knative.default.127.0.0.1.xip.io` we will see a pod is started

```bash
$ kubectl get pods
NAME                                                   READY   STATUS    RESTARTS   AGE
hello-world-knative-00001-deployment-548f9f94c-zm4cp   2/2     Running   0          18s
```

and also the `Active` conditions status is changed

```bash
Conditions:
  OK TYPE                  AGE REASON
  ++ Ready                 58m
  ++ ContainerHealthy      58m
  ++ ResourcesAvailable    58m
  ++ Active                23s
```

By default, the `Autoscaler` makes a decision based on the past 60 seconds, so if there is no traffic the pod is terminated.

The `Configuration` object is not exposed by the `kn` CLI, we can access it using `kubectl`

```bash
$ kubectl get config
NAME                  LATESTCREATED               LATESTREADY                 READY   REASON
hello-world-knative   hello-world-knative-00001   hello-world-knative-00001   True

$ kubectl describe config hello-world-knative
Name:         hello-world-knative
Namespace:    default
Labels:       serving.knative.dev/service=hello-world-knative
              serving.knative.dev/serviceUID=87232544-1140-464d-a1ef-421cf38a3916
Annotations:  serving.knative.dev/creator: docker-for-desktop
              serving.knative.dev/lastModifier: docker-for-desktop
              serving.knative.dev/routes: hello-world-knative
API Version:  serving.knative.dev/v1
Kind:         Configuration
Metadata:
  Creation Timestamp:  2021-03-14T20:02:27Z
  Generation:          1
  Managed Fields:
    API Version:  serving.knative.dev/v1
    Fields Type:  FieldsV1
    Manager:    controller
    Operation:  Update
    Time:       2021-03-14T20:02:37Z
Spec:
  Template:
    Spec:
      Containers:
        Env:
          Name:   TARGET
          Value:  Knative
        Image:    docker.io/altfatterz/hello-world-knative:0.0.1-SNAPSHOT    
Status:
  Conditions:
    Last Transition Time:        2021-03-14T20:02:37Z
    Status:                      True
    Type:                        Ready
  Latest Created Revision Name:  hello-world-knative-00001
  Latest Ready Revision Name:    hello-world-knative-00001
  Observed Generation:           1
Events:
  Type    Reason              Age   From                      Message
  ----    ------              ----  ----                      -------
  Normal  Created             17m   configuration-controller  Created Revision "hello-world-knative-00001"
  Normal  ConfigurationReady  17m   configuration-controller  Configuration becomes ready
  Normal  LatestReadyUpdate   17m   configuration-controller  LatestReadyRevisionName updated to "hello-world-knative-00001"
```

Let's update the service by changing the value of the `TARGET` environment variable.

```bash
$ kn service update hello-world-knative --image=docker.io/altfatterz/hello-world-knative:0.0.1-SNAPSHOT --env TARGET="Knative community"
```

Under the hood, the `kn` CLI updates the YAML document and sends it to the Kubernetes API server.
We see that a new `Revision` is created and the existing `Configuration` is pointing to this newly created `Revision`. There is a parent-child relationship between a `Configuration` and a `Revision`.  
We can also use the `kubectl` CLI to access the Knative objects:

```bash
$ kubectl get revisions
NAME                        CONFIG NAME           K8S SERVICE NAME            GENERATION   READY   REASON
hello-world-knative-00001   hello-world-knative   hello-world-knative-00001   1            True
hello-world-knative-00002   hello-world-knative   hello-world-knative-00002   2            True
```

```bash
$ kubectl get configurations
NAME                  LATESTCREATED               LATESTREADY                 READY   REASON
hello-world-knative   hello-world-knative-00002   hello-world-knative-00002   True
```


### Running on Cloud Run

[Cloud Run](https://cloud.google.com/run) is one of the fully managed [Knative offerings](https://knative.dev/docs/knative-offerings/) available which we are going to use in this blog post.
It implements most parts of the Knative Serving API. There is another version [Cloud Run for Anthos](https://cloud.google.com/anthos/run) which provides Knative installation on your existing Kubernetes/GKE cluster. 

<p><img src="/images/2021-03-14/CloudRun.png" alt="CloudRun" /></p>
   
First let's configure our environment with `gcloud` the [Google Cloud SDK](https://cloud.google.com/sdk/docs/install): extract the Google Cloud project id into an environment variable (to be referenced later), enable the `Cloud Run` API, set the `Cloud Run` version to be managed, and set a default region for `Cloud Run`. 

```
$ PROJECT_ID=$(gcloud config get-value project)
$ gcloud services enable run.googleapis.com
$ gcloud config set run/platform managed
$ gcloud config set run/region europe-west6
```

Using `Cloud Run` we can deploy a container image stored in [Container Registry](https://cloud.google.com/container-registry) or [Artifact Registry](https://cloud.google.com/artifact-registry). We are going to use here `Container Registry`.
First, we tag our image:

```bash
$ docker tag altfatterz/hello-world-knative:0.0.1-SNAPSHOT eu.gcr.io/${PROJECT_ID}/hello-world-knative:0.0.1-SNAPSHOT
```

Next, we configure docker authentication for `gcloud` and push the image:

```bash
$ gcloud auth configure-docker
$ docker push eu.gcr.io/${PROJECT_ID}/hello-world-knative:0.0.1-SNAPSHOT
``` 

We can verify that the image was uploaded using:

```bash
$ gcloud container images list-tags eu.gcr.io/${PROJECT_ID}/hello-world-knative
DIGEST        TAGS            TIMESTAMP
1f2a539da7d5  0.0.1-SNAPSHOT  2021-03-14T08:40:25
```

Finally, we deploy our service: 

```bash
$ gcloud run deploy hello-world-knative --allow-unauthenticated \
  --cpu=2 --memory=1G --set-env-vars="SPRING_PROFILES_ACTIVE=prod" \
  --image=eu.gcr.io/${PROJECT_ID}/hello-world-knative:0.0.1-SNAPSHOT
```

After a minute or so we have our service deployed:

```bash
Deploying container to Cloud Run service [hello-world-knative] in project [cloud-run-demos-306217] region [europe-west6]
✓ Deploying new service... Done.
  ✓ Creating Revision...
  ✓ Routing traffic...
  ✓ Setting IAM Policy...
Done.
Service [hello-world-knative] revision [hello-world-knative-00001-pet] has been deployed and is serving 100 percent of traffic.
Service URL: https://hello-world-knative-edbd2pdvbq-oa.a.run.app
```

For more information look into the official documentation of [Cloud Run](https://cloud.google.com/run/docs). The [cloud-run-faq](https://github.com/ahmetb/cloud-run-faq) community-maintained informal knowledge base is also very useful. 

### Conclusion

We have seen how to set up a basic local `Knative Serving` environment, deploy a [simple](https://github.com/altfatterz/cloud-run-demos/tree/master/hello-world-knative) Spring Boot application on it. We have looked into the core concepts of `Knative Serving` and also we have seen how we can deploy our application to a fully managed `Knative Serving` offering like `Cloud Run` from Google   
