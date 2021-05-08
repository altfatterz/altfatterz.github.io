---
layout: post
title: Local development with OpenShift
tags: [openshift, container]
---

Requirements:

- 4 virtual CPUs (vCPUs)
- 8 GB of memory
- 35 GB of storage space

```bash
$ crc setup

INFO Checking if podman remote executable is cached
INFO Checking if goodhosts executable is cached
INFO Caching goodhosts executable
INFO Will use root access: change ownership of /Users/altfatterz/.crc/bin/goodhosts
Password:
INFO Will use root access: set suid for /Users/altfatterz/.crc/bin/goodhosts
INFO Checking if CRC bundle is extracted in '$HOME/.crc'
INFO Extracting bundle from the CRC executable
crc.qcow2: 394.75 MiB / 10.16 GiB [----------->] 100.00%
INFO Checking minimum RAM requirements
INFO Checking if running as non-root
INFO Checking if HyperKit is installed
INFO Setting up virtualization with HyperKit
INFO Will use root access: change ownership of /Users/altfatterz/.crc/bin/hyperkit
INFO Will use root access: set suid for /Users/altfatterz/.crc/bin/hyperkit
INFO Checking if crc-driver-hyperkit is installed
INFO Installing crc-machine-hyperkit
INFO Will use root access: change ownership of /Users/altfatterz/.crc/bin/crc-driver-hyperkit
INFO Will use root access: set suid for /Users/altfatterz/.crc/bin/crc-driver-hyperkit
INFO Checking file permissions for /etc/hosts
INFO Checking file permissions for /etc/resolver/testing
INFO Setting file permissions for /etc/resolver/testing
INFO Will use root access: create dir /etc/resolver
INFO Will use root access: create file /etc/resolver/testing
INFO Will use root access: change ownership of /etc/resolver/testing
Setup is complete, you can now run 'crc start' to start the OpenShift cluster

```

We set the memory to 12 GB (the default is set to 9BG) in order to be able to enable the `openshift-monitoring` operator. Otherwise, the operator would not start.  

```bash
$ crc config set memory 12228
```

```bash
$ crc start

INFO Checking if podman remote executable is cached
INFO Checking if goodhosts executable is cached
INFO Checking minimum RAM requirements
INFO Checking if running as non-root
INFO Checking if HyperKit is installed
INFO Checking if crc-driver-hyperkit is installed
INFO Checking file permissions for /etc/hosts
INFO Checking file permissions for /etc/resolver/testing
? Image pull secret [? for help] ***************************************************************************************
INFO Verifying validity of the kubelet certificates ...
INFO Starting OpenShift kubelet service
INFO Configuring cluster for first start
INFO Adding user's pull secret to the cluster ...
INFO Updating cluster ID ...
INFO Starting OpenShift cluster ... [waiting 3m]
INFO Updating kubeconfig
WARN The cluster might report a degraded or error state. This is expected since several operators have been disabled to 
lower the resource usage. For more information, please consult the documentation

Started the OpenShift cluster

To access the cluster, first set up your environment by following the instructions returned by executing 'crc oc-env'.
Then you can access your cluster by running 'oc login -u developer -p developer https://api.crc.testing:6443'.
To login as a cluster admin, run 'oc login -u kubeadmin -p HqC3I-wgtiB-q7qCf-KEsuK https://api.crc.testing:6443'.

You can also run 'crc console' and use the above credentials to access the OpenShift web console.
The console will open in your default browser.
```


```bash
$ crc console
```

<p><img src="/images/2020-12-30/openshift-console.png" alt="OpenShift Console" /></p>

To display status of the OpenShift cluster

```bash
$ crc status

CRC VM:          Running
OpenShift:       Running (v4.6.6)
Disk Usage:      11.38GB of 32.72GB (Inside the CRC VM)
Cache Usage:     13.87GB
Cache Directory: /Users/altfatterz/.crc/cache
```

The [documentation](https://docs.openshift.com/container-platform/4.6/cli_reference/openshift_cli/getting-started-cli.html#installing-the-cli) suggests to download the binary, but with using Homebrew is much easier to manage: 

```bash
$ brew install openshift-cli
```

Another way to setup the `oc` command line tool is with `crc`

```bash
eval $(crc oc-env)
``` 

To access the cluster you need to first login using the credentials printed at the end of the logs of the `crc start` command. 
However, if you cleared your screen you can access them again using:

```bash
$ crc console --credentials
``` 

To check the cluster status we can use:

```bash
$ crc status
```

Another way is with is the `oc` comand line tool:

```bash
$ oc status
In project default on server https://api.crc.testing:6443
svc/openshift - kubernetes.default.svc.cluster.local
svc/kubernetes - 172.25.0.1:443 -> 6443
```

To get the nodes:

```bash
$ oc get nodes
NAME                 STATUS   ROLES           AGE   VERSION
crc-fdm75-master-0   Ready    master,worker   27d   v1.19.0+43983cd
```

To debug a node we can use:

```bash
$ oc debug node/crc-fdm75-master-0
```

This will spin up a debug pod (in another terminal you can check `oc get pods`) which can be used to figure out what is going on with our node. 



```bash
$ oc get clusterversion version -ojsonpath='{range .spec.overrides[*]}{.name}{"\n"}{end}' | nl -v 0 
``` 
 
Start monitoring, alerting, and telemetry services:

```bash
$ oc patch clusterversion/version --type='json' -p '[{"op":"remove", "path":"/spec/overrides/0"}]'  
```
 
```bash
$ oc project openshift-monitoring
```

```bash
$ oc get pods

TODO with a list
```
 
 
When we are ready with the demo we can stop and delete our cluster using:

```bash
$ crc stop
$ crc delete
```

Resources:
 
1. Getting Started Guide with RedHat CodeReady Containers 
https://code-ready.github.io/crc/
 
2. S2I BuildConfigs on OpenShift
https://tomd.xyz/openshift-s2i-example/ 

3. 