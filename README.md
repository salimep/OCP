# OpenShift 4.14.12 Bare Metal Install - User Provisioned Infrastructure (UPI) on restricted/disconnected enviorment 

- [OpenShift 4 Bare Metal Install - User Provisioned Infrastructure (UPI)](#openshift-4-bare-metal-install---user-provisioned-infrastructure-upi)
  - [Architecture Diagram](#architecture-diagram)
  - [Download Software](#download-software)
  - [Setting up local-registry](#configure-local-registry)
  - [Generate and host install files](#generate-and-host-install-files)
  - [Deploy OpenShift](#deploy-openshift)
  - [Monitor the Bootstrap Process](#monitor-the-bootstrap-process)
  - [Remove the Bootstrap Node](#remove-the-bootstrap-node)
  - [Wait for installation to complete](#wait-for-installation-to-complete)
  - [Join Worker Nodes](#join-worker-nodes)
  - [Access the OpenShift Console](#access-the-openshift-console)

## Architecture Diagram


## Download Software

1. Download [CentOS 8 x86_64 image](https://www.centos.org/centos-linux/)
1. Login to [RedHat OpenShift Cluster Manager](https://cloud.redhat.com/openshift)
1. Select 'Create Cluster' from the 'Clusters' navigation menu
1. Select 'RedHat OpenShift Container Platform'
1. Select 'Run on Bare Metal'
1. Download the following files:

   - Openshift Installer for Linux
   - Pull secret
   - Command Line Interface for Linux and your workstations OS
   - Red Hat Enterprise Linux CoreOS (RHCOS)
     - rhcos-X.X.X-x86_64-metal.x86_64.raw.gz
     - rhcos-X.X.X-x86_64-installer.x86_64.iso (or rhcos-X.X.X-x86_64-live.x86_64.iso for newer versions)

configure local registry with UI [Setting up local-registry](#configure-local-registry)
1)	Generate CA & certificate for your docker container
## ca creation
```bash
openssl genrsa -out CA.key  2048
openssl req -x509 -new -nodes -key myCA.key -sha256 -days 1825 -out myCA.pem
```

# docker certificate

```bash
genrsa -out docker.key 2048
openssl x509 -req -in docker.csr -signkey docker.key -out docker.crt -CA ../myCA.pem -CAkey ../myCA.key -days 365 -extensions req_ext -extfile ssl.cnf
```
Create htpasswd file for basic authentication
```bash
htpasswd -cb  htpasswd admin mysecret
```
 Trust your custom ca all your infra nodes 
 ```bash
sudo cp myCA.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```          
Load docker image registry in disconnected environment and spin the container
```bash
podman load -i docker-registry.tar
podman load -i docker-registry-ui.tar
## run UI (optional)

podman run --name registry-ui -p 8080:80 -e SINGLE_REGISTRY=true -e REGISTRY_TITLE=Docker Registry UI -e DELETE_IMAGES=true -e SHOW_CONTENT_D
IGEST=true -e NGINX_PROXY_PASS_URL=https://local-registry:5000 -e SHOW_CATALOG_NB_TAGS=true -e CATALOG_MIN_BRANCHES=1 -e CATALOG_MAX_BRANCHES=1 -e C
ATAGLIST_PAGE_SIZE=100 -e REGISTRY_SECURED=false -e CATALOG_ELEMENTS_LIMIT=1000 -d docker-registry-ui:main

## run registry

podman run --name local-registry -p 192.168.10.110:5000:5000 -v /opt/registry/data:/var/lib/registry:z -v /opt/registry/auth:/auth:z -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v /opt/registry/certs:/certs:z -v /opt/registry/registry_config.yml:/etc/docker/registry/config.yml -e "REGISTRY_HTTP_TLS_CERTIFICATE=/certs/docker.crt" -e "REGISTRY_HTTP_TLS_KEY=/certs/docker.key" -e REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED=true -d registry:2.8.2
```
## Download ocp artifacts  and mirror the image
https://cloud.redhat.com/openshift
```bash
OCP_RELEASE=4.14.12
ARCHITECTURE=x86_64
LOCAL_REPOSITORY=ocp-release/openshift4
PRODUCT_REPO=openshift-release-dev
LOCAL_SECRET_JSON=/root/pull.json
RELEASE_NAME=ocp-release

echo "Sourcing ImageContentPolicy for this release"
oc adm -a ${LOCAL_SECRET_JSON} release mirror --from quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
        --to-dir /root/mirror/ocp-release
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.12/openshift-client-linux.tar.gz

wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.12/openshift-install-linux.tar.gz

wget  https://mirror.openshift.com/pub/openshiftv4/x86_64/dependencies/rhcos/4.12/4.12.0/rhcos-4.12.0-x86_64-metal.x86_64.raw.gz

wget  https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.12/4.12.0/rhcos-4.12.0-x86_64-live.x86_64.iso

 ```
tar it all downloaded files and send to disconnected enviorment

Uploading  opensshift images into local repository

```bash
 oc image mirror --from-dir=/root/mirror/ocp-release "file://openshift/release:${OCP_RELEASE}*" local-registry:5000/ocp-release/openshift4 --keep-manifest-list=true 
```
## Generate and host install files


1. Generate an SSH key pair keeping all default options

   ```bash
   ssh-keygen
   ```

1. Create an install directory

   ```bash
   mkdir ~/ocp-install
   ```

1. Copy the install-config.yaml included in the clones repository to the install directory

   ```bash
   cp install-config.yaml ~/ocp-install
   ```

1. Update the install-config.yaml with your own pull-secret and ssh key.

   - Line 23 should contain the contents of your pull-secret.txt
   - append your localregistry authentication into pull-secret file well ( its very importent otherwise it will not authenticate with your local registry repositories)
   - Line 24 should contain the contents of your '~/.ssh/id_rsa.pub'
   - paste your custom CA file which we created part of docker-registry setup
   - append your localregistry url inside the install-config.yaml as shown in below
   - 
```bash
   vim ~/ocp-install/install-config.yaml
```
```yaml
additionalTrustBundle: |
        -----BEGIN CERTIFICATE-----
        XXXXXXXXXXXXXXX
        -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - local-registry:5000/ocp-release/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - local-registry:5000/ocp-release/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```


1. Generate Kubernetes manifest files

   ```bash
   ~/openshift-install create manifests --dir ~/ocp-install
   ```

   > A warning is shown about making the control plane nodes schedulable. It is up to you if you want to run workloads on the Control Plane nodes. If you dont want to you can disable this with:
   > `sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' ~/ocp-install/manifests/cluster-scheduler-02-config.yml`.
   > Make any other custom changes you like to the core Kubernetes manifest files such as NTP config
   ```bash
   cp 99-master-chrony.yaml 99-worker-chrony.yaml ~/ocp-install/manifests/
   ```
   Generate the Ignition config and Kubernetes auth files

   ```bash
   ~/openshift-install create ignition-configs --dir ~/ocp-install/
   ```

1. Create a hosting directory to serve the configuration files for the OpenShift booting process

   ```bash
   mkdir /var/www/html/ocp4
   ```

1. Copy all generated install files to the new web server directory

   ```bash
   cp -R ~/ocp-install/* /var/www/html/ocp4
   ```

1. Move the Core OS image to the web server directory (later you need to type this path multiple times so it is a good idea to shorten the name)

   ```bash
   mv ~/rhcos-X.X.X-x86_64-metal.x86_64.raw.gz /var/www/html/ocp4/rhcos
   ```

1. Change ownership and permissions of the web server directory

   ```bash
   chcon -R -t httpd_sys_content_t /var/www/html/ocp4/
   chown -R apache: /var/www/html/ocp4/
   chmod 755 /var/www/html/ocp4/
   ```

1. Confirm you can see all files added to the `/var/www/html/ocp4/` dir through Apache

   ```bash
   curl localhost:8080/ocp4/
   ```

## Deploy OpenShift

1. Power on the ocp-bootstrap host and ocp-cp-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:

   ```bash
   # Bootstrap Node - ocp-bootstrap
   coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.10.11:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.10.11:8080/ocp4/bootstrap.ign
   
   # Or if you waited for it boot, use the following command then just reboot after it finishes and make sure you remove the attached .iso
   sudo coreos-installer install /dev/sda -u http://192.168.10.11:8080/ocp4/rhcos -I http://192.168.10.11:8080/ocp4/bootstrap.ign --insecure --insecure-ignition
   ```

   ```bash
   # Each of the Control Plane Nodes - ocp-cp-\#
   coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.10.11:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.10.11:8080/ocp4/master.ign
   
   # Or if you waited for it boot, use the following command then just reboot after it finishes and make sure you remove the attached .iso
   sudo coreos-installer install /dev/sda -u http://192.168.10.11:8080/ocp4/rhcos -I http://192.168.10.11:8080/ocp4/master.ign --insecure --insecure-ignition
   ```

1. Power on the ocp-w-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:

   ```bash
   # Each of the Worker Nodes - ocp-w-\#
   coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.10.11:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.10.11:8080/ocp4/worker.ign
   
   # Or if you waited for it boot, use the following command then just reboot after it finishes and make sure you remove the attached .iso
   sudo coreos-installer install /dev/sda -u http://192.168.10.11:8080/ocp4/rhcos -I http://192.168.10.11:8080/ocp4/worker.ign --insecure --insecure-ignition
   ```

## Monitor the Bootstrap Process

1. You can monitor the bootstrap process from the ocp-svc host at different log levels (debug, error, info)

   ```bash
   ~/openshift-install --dir ~/ocp-install wait-for bootstrap-complete --log-level=debug
   ```

1. Once bootstrapping is complete the ocp-boostrap node [can be removed](#remove-the-bootstrap-node)

## Remove the Bootstrap Node

1. Remove all references to the `ocp-bootstrap` host from the `/etc/haproxy/haproxy.cfg` file

   ```bash
   # Two entries
   vim /etc/haproxy/haproxy.cfg
   # Restart HAProxy - If you are still watching HAProxy stats console you will see that the ocp-boostrap host has been removed from the backends.
   systemctl reload haproxy
   ```

1. The ocp-bootstrap host can now be safely shutdown and deleted from the VMware ESXi Console, the host is no longer required

## Wait for installation to complete

> IMPORTANT: if you set mastersSchedulable to false the [worker nodes will need to be joined to the cluster](#join-worker-nodes) to complete the installation. This is because the OpenShift Router will need to be scheduled on the worker nodes and it is a dependency for cluster operators such as ingress, console and authentication.

1. Collect the OpenShift Console address and kubeadmin credentials from the output of the install-complete event

   ```bash
   ~/openshift-install --dir ~/ocp-install wait-for install-complete
   ```

1. Continue to join the worker nodes to the cluster in a new tab whilst waiting for the above command to complete

## Join Worker Nodes

1. Setup 'oc' and 'kubectl' clients on the ocp-svc machine

   ```bash
   export KUBECONFIG=~/ocp-install/auth/kubeconfig
   # Test auth by viewing cluster nodes
   oc get nodes
   ```

1. View and approve pending CSRs

   > Note: Once you approve the first set of CSRs additional 'kubelet-serving' CSRs will be created. These must be approved too.
   > If you do not see pending requests wait until you do.

   ```bash
   # View CSRs
   oc get csr
   # Approve all pending CSRs
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   # Wait for kubelet-serving CSRs and approve them too with the same command
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   ```

1. Watch and wait for the Worker Nodes to join the cluster and enter a 'Ready' status

   > This can take 5-10 minutes

   ```bash
   watch -n5 oc get nodes
   ```

## Access the OpenShift Console

1. Wait for the 'console' Cluster Operator to become available

   ```bash
   oc get co
   ```

1. Append the following to your local workstations `/etc/hosts` file:

   > From your local workstation
   > If you do not want to add an entry for each new service made available on OpenShift you can configure the ocp-svc DNS server to serve externally and create a wildcard entry for \*.apps.lab.ocp.lan

   ```bash
   # Open the hosts file
   sudo vi /etc/hosts

   # Append the following entries:
   192.168.0.96 ocp-svc api.lab.ocp.lan console-openshift-console.apps.lab.ocp.lan oauth-openshift.apps.lab.ocp.lan downloads-openshift-console.apps.lab.ocp.lan alertmanager-main-openshift-monitoring.apps.lab.ocp.lan grafana-openshift-monitoring.apps.lab.ocp.lan prometheus-k8s-openshift-monitoring.apps.lab.ocp.lan thanos-querier-openshift-monitoring.apps.lab.ocp.lan
   ```

1. Navigate to the [OpenShift Console URL](https://console-openshift-console.apps.cloud.salimonline.local) and log in as the 'admin' user

   > You will get self signed certificate warnings that you can ignore
   > If you need to login as kubeadmin and need to the password again you can retrieve it with: `cat ~/ocp-install/auth/kubeadmin-password`

## Troubleshooting

1. You can collect logs from all cluster hosts by running the following command from the 'ocp-svc' host:

   ```bash
   ./openshift-install gather bootstrap --dir ocp-install --bootstrap=192.168.10.200 --master=192.168.10.201 --master=192.168.10.202 --master=192.168.10.203
   ```

1. Modify the role of the Control Plane Nodes

   If you would like to schedule workloads on the Control Plane nodes apply the 'worker' role by changing the value of 'mastersSchedulable' to true.

   If you do not want to schedule workloads on the Control Plane nodes remove the 'worker' role by changing the value of 'mastersSchedulable' to false.

   > Remember depending on where you host your workloads you will have to update HAProxy to include or exclude the control plane nodes from the ingress backends.

   ```bash
   oc edit schedulers.config.openshift.io cluster
   ```
