# OpenShift 4.14.12 image registry setup for disconnected enviorenment 

-  [create PV for image storage](#)
  - [configure image-registry config ](#architecture-diagram)
  - [get route from openshift-image-registry NS ](#download-software)
  - [login to openshift image registry](#configure-local-registry)
  - [pull the image into registry](#generate-and-host-install-files)

create PV for nfs storage assume you  already have  nfs-server in your enviorenment
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-pv
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 100Gi
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /shares/registry
    server: 192.168.10.30
    mountOptions:
    - hard
    - nfsvers=4
```
```bash
oc apply -f pv.yml
```

modify image registry config

```bash

oc edit configs.imageregistry.operator.openshift.io
```
modify the spec as shown in below 

```yaml

spec:
  defaultRoute: true ## to create default route
  httpSecret: ##############
  logLevel: Normal
  managementState: Managed ## 
  observedConfig: null
  operatorLogLevel: Normal
  proxy: {}
  replicas: 1
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  rolloutStrategy: RollingUpdate
  storage:
    managementState: Managed
    pvc:
      claim: image-registry-storage ## PVC name
  unsupportedConfigOverrides: null
```
PVC yaml Example
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeMode: Filesystem
  volumeName: registry-pv
```
check image-registry-pod status ( all pod shoud be running state)
```bash
[root@infra ~]# oc get pod -n openshift-image-registry
NAME                                               READY   STATUS    RESTARTS   AGE
cluster-image-registry-operator-7dd75c64c9-mv9dc   1/1     Running   0          41m
image-registry-787f8f6cf6-fxvrd                    1/1     Running   0          41m
node-ca-4q7zv                                      1/1     Running   0          41m
node-ca-7cnhc                                      1/1     Running   0          41m
node-ca-chdh2                                      1/1     Running   0          41m
node-ca-gw66t                                      1/1     Running   0          41m
node-ca-mw979                                      1/1     Running   0          41m
[root@infra ~]# 
```
get route for login

```bash
[root@infra ~]# oc get route -n openshift-image-registry
NAME            HOST/PORT                                                               PATH   SERVICES         PORT    TERMINATION   WILDCARD
default-route   default-route-openshift-image-registry.apps.cluster.salimonline.local          image-registry   <all>   reencrypt     None
[root@infra ~]# 

```
finally login to image registry

```bash
[root@infra ~]# podman login -u salim -p $(oc whoami -t)  default-route-openshift-image-registry.apps.cluster.salimonline.local --tls-verify=false
Login Succeeded!

```
pull the image  into it
```bash
[root@infra ~]# podman images
REPOSITORY  TAG         IMAGE ID    CREATED     SIZE
[root@infra ~]# podman load -i ose-cli.tar 
Getting image source signatures
Copying blob d3fbfed1573d done  
Copying blob deca8fff2db8 done  
Copying blob 2a6e5e6e95f7 done  
Copying blob b1bd991545cc done  
Copying blob e2e51ecd22dc done  
Copying config 90e7023f68 done  
Writing manifest to image destination
Loaded image: registry.redhat.io/openshift4/ose-cli:v4.12.0-202411040131.p0.gd691257.assembly.stream.el8
[root@infra ~]# podman images
REPOSITORY                             TAG                                                   IMAGE ID      CREATED      SIZE
registry.redhat.io/openshift4/ose-cli  v4.12.0-202411040131.p0.gd691257.assembly.stream.el8  90e7023f6843  2 weeks ago  594 MB

```
