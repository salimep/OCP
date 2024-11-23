# upgrade OpenShift cluster from 4.14.12 to 4.14.13 on disconnected environment 

-  [mirror the new image from redhat registry](#)
  - [upload mirror image into local registry ](#architecture-diagram)
  - [apply the the signature configmap](#download-software)
  - [upgrade OCP cluster](#configure-local-registry)

mirror the image  from register.redhat.io ( from your lap which have reliable internet)

```bash
export OCP_RELEASE=4.14.13
export ARCHITECTURE=x86_64
export LOCAL_REGISTRY=local-registry-2:5000
export LOCAL_REPOSITORY=ocp-release/openshift4
export PRODUCT_REPO=openshift-release-dev
export LOCAL_SECRET_JSON=/root/pull.json
export RELEASE_NAME=ocp-release

echo "Sourcing ImageContentPolicy for this release"
oc adm -a ${LOCAL_SECRET_JSON} release mirror --from quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
	--to-dir=/root/mirror/ocp-release --to ${LOCAL_REGISTRY}/openshift --to-release-image=${LOCAL_REGISTRY}/openshift/release-image \
	--dry-run > /root/mirror/icsp.txt

echo "Mirroring  containers for OCP ${OCP_RELEASE}"

oc adm -a ${LOCAL_SECRET_JSON} release mirror --from quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
	--to-dir /root/mirror/ocp-release

wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_RELEASE}/openshift-install-linux-${OCP_RELEASE}.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_RELEASE}/openshift-client-linux.tar.gz
```
tar the /root/mirror/ocp-release  directory and upload into disconnected enviorment 
login to disconnected MGM node

and set variables and upload into your localregistry

```bash
export OCP_RELEASE=4.14.13
export  ARCHITECTURE=x86_64
export LOCAL_REGISTRY=local-registry-2:5000
export LOCAL_REPOSITORY=ocp-release/openshift4
export PRODUCT_REPO=openshift-release-dev
export LOCAL_SECRET_JSON=/root/pull.json
export RELEASE_NAME=ocp-release

tar xzvf xxxxx.tar.gz -C /home/

podman login --authfile ~/.docker/config.json local-registry-2.salimonline.local:5000

oc image mirror --from-dir=/home/root/mirror/ocp-release "file://openshift/release:4.14.13*" local-registry-2.salimonline.local:5000/ocp-release/openshift4 --keep-manifest-list=true

```

once mirror is completed verify the the signatrue sha value with openshift-install quay image 

```bash

[root@infra config]# pwd
/home/root/mirror/ocp-release/config
[root@infra config]# cat signature-sha256-ff004c122db9d494.json 
{"kind":"ConfigMap","apiVersion":"v1","metadata":{"name":"sha256-ff004c122db9d494abedc1630ef5e2e535cc4529b3450ff58aaacf6b6cbba30c","namespace":"openshift-config-managed","creationTimestamp":null,"labels":{"release.openshift.io/verification-signatures":""}},"binaryData":{"sha256-ff004c122db9d494abedc1630ef5e2e535cc4529b3450ff58aaacf6b6cbba30 .........

```
to makesure you have 14.13 release image(sha256-ff004c122db9d494...)  verify with openshift-install client 

```bash
[root@infra home]# ./openshift-install version
./openshift-install 4.14.13
built from commit 6ad9135c45355a6f0bcc0899c538ba594b6f26f2
release image quay.io/openshift-release-dev/ocp-release@sha256:ff004c122db9d494abedc1630ef5e2e535cc4529b3450ff58aaacf6b6cbba30c
release architecture amd64
```
apply the signature confimap

```bash
cd /home/root/mirror/ocp-release/config
oc apply -f signature-sha256-ff004c122db9d494.json
```

upgrade OCP cluster to 4.14.12 to 4.14.13

```bash
oc adm upgrade --allow-explicit-upgrade --to-image quay.io/openshift-release-dev/ocp-release@sha256:ff004c122db9d494abedc1630ef5e2e535cc4529b3450ff58aaacf6b6cbba30c
```
check OCP Upgrade Process STATUS ( will notify the progress)
```bash 
[root@infra home]# oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.14.13   True        False         16h     Cluster version is 4.14.13
```

