---
layout: post
title: Setup Minishift/Openshift 
tags: [minishift, openshift, kubernetes, docker]
---

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
