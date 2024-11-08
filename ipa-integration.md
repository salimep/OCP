# OpenShift Redhat IPA integration guide

-  [create secret for ipa user](#)
  - [create configmap for IPA ca](#architecture-diagram)
  - [modify oauth yaml](#download-software)
  - [apply yaml and check the pod status](#configure-local-registry)



verify user credentials using ldapsearch 

```bash
ldapsearch -D "cn=salim"  -w xxxx -H ldaps://ipa.salimonline.local ## this user should have rights  to see other uid's
```


[create secret for ipa user]
```bash
oc create secret generic ipa-ldap-secret --from-literal bindPassword=xxx -n openshift-config
```
create configmap for IPA ca

```bash
oc create configmap ipa-ca-config-map --from-file ca.crt=ca.crt -n openshift-config
```
modify oauth yaml file
```bash
oc get oauth cluster -o yaml > oauth.yml
```
```yaml

apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - ldap:
      attributes:
        id:
          - dn
        email:
          - mail
        name:
          - cn
        preferredUsername:
          - uid
      bindDN: 'cn=salim'
      bindPassword:
        name: ipa-ldap-secret
      ca:
        name: ipa-ca-config-map
      insecure: false
      url: >-
        ldaps://ipa.salimonline.local/dc=salimonline,dc=local?uid
    mappingMethod: claim
    name: Redhat IPA
    type: LDAP
```
apply the modified oauth file
```bash
oc apply -f oauth.yml

status
--
oc -n openshift-authentication get pod #( its hould be recreated)
```          
