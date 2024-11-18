# OpenShift 4.14.12 Bare Metal MC for NTP service

-  [create butane configuration](#)
  - [convert butane image to MC yaml](#architecture-diagram)
  - [apply MC yaml](#download-software)
  - [check MCP status](#configure-local-registry)



[create butane config]
```yaml
variant: openshift
version: 4.12.0
metadata:
  name: 99-worker-custom
  labels:
    machineconfiguration.openshift.io/role: master
openshift:
  kernel_arguments:
    - loglevel=7
storage:
  files:
    - path: /etc/chrony.conf
      mode: 0644
      overwrite: true
      contents:
        inline: |
          server 192.168.10.11 iburst
          driftfile /var/lib/chrony/drift
          makestep 1.0 3
          rtcsync
          logdir /var/log/chrony
```
install butane binary & convert to MC yaml for mac
```bash
brew install butane

butane 99-master-chrony.bu -o 99-master-chrony.yml
```
generated MC ( MachineConfig) yaml

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-worker-custom
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - contents:
            compression: ""
            source: data:,server%20192.168.10.11%20iburst%0Adriftfile%20%2Fvar%2Flib%2Fchrony%2Fdrift%0Amakestep%201.0%203%0Artcsync%0Alogdir%20%2Fvar%2Flog%2Fchrony%0A
          mode: 420
          overwrite: true
          path: /etc/chrony.conf
  kernelArguments:
    - loglevel=7
```
apply MC yaml
```bash
oc apply -f 99-master-chrony.yml

```
check MCP ( machineConfigPool) status 
```bash
oc get mcp #(UPDATING column should be True and ID shoudl match with MC ID to verify)

NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-bb156e50b92df989e079a74dc1a48edc   False     True       False      3              0                   0                     0                      7d21h
worker   rendered-worker-4d5d081553296c1e71fea5ea47040f66   False     True       False      2              0                   0                     0                      7d21h
 incase if you apply by force

```bash
oc patch machineconfigpool master --type=merge -p '{"metadata":{"annotations":{"machineconfiguration.openshift.io/desiredConfig":"100-worker-custom-ntp"}}}'
```
