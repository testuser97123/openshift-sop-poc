## openShift deployment binaries ##

- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.18.12/oc-mirror.tar.gz
- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.18.12/openshift-client-linux-4.18.12.tar.gz
- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.18.12/openshift-install-linux-4.18.12.tar.gz
- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.18.12/opm-linux-4.18.12.tar.gz

- https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/mirror-registry/1.3.11/mirror-registry.tar.gz

- https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/helm/3.15.4/helm-linux-amd64


- https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/butane-amd64

- https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.18/latest/rhcos-4.18.1-x86_64-live.x86_64.iso

- https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.18/latest/rhcos-4.18.1-x86_64-nutanix.x86_64.qcow2

- https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.18/latest/rhcos-4.18.1-x86_64-vmware.x86_64.ova


## Extract the oc-mirror binary. 

```yml
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.18.12/oc-mirror.tar.gz

tar xf oc-mirror.tar.gz -C /usr/bin ; chmod 777 /usr/bin/oc-mirror 
```


### create a install-config.yaml 

```yml
    apiVersion: v1
    baseDomain: office.ocpztp.com
    compute:
    - architecture: amd64
      hyperthreading: Enabled
      name: worker
      replicas: 3
    controlPlane:
      architecture: amd64
      hyperthreading: Enabled
      name: master
      replicas: 3
    metadata:
      name: orbocp
    networking:
      clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
      networkType: OVNKubernetes
      serviceNetwork:
      - 172.30.0.0/16
    platform:
      none: {}
    fips: false
    pullSecret: ''
    sshKey: ''
    imageContentSources:
    - mirrors:
      - kvmhost-raven.blackbird.com:8443/openshift4/ocp-release
      source: quay.io/openshift-release-dev/ocp-release
    - mirrors:
      - kvmhost-raven.blackbird.com:8443/openshift4/ocp-release
      source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```    

### Mirror to Mirror vanilla OpenShift Images

```yml
oc adm release mirror --from=quay.io/openshift-release-dev/ocp-release:4.18.12-x86_64 --to=kvmhost-raven.blackbird.com:8443/openshift4/ocp-release --to-release-image=kvmhost-raven.blackbird.com:8443/openshift4/ocp-release:4.18.12
```
or 

### Mirror to Disk OCP Base Images  
```yml
vim openshift-vars.sh 
export OCP_RELEASE=4.18.12
export LOCAL_REGISTRY=kvmhost-raven.blackbird.com:8443
export LOCAL_REPOSITORY=ocp4/openshift4
export REG_CREDS=/root/quay-secret.json
export PRODUCT_REPO=openshift-release-dev
export RELEASE_NAME=ocp-release
export ARCHITECTURE=x86_64
export GODEBUG=x509ignoreCN=0
export REMOVABLE_MEDIA_PATH=/ocpregistry/ocp-base-images

source openshift-vars.sh 
echo $LOCAL_REGISTRY
kvmhost-raven.blackbird.com:8443
```
### Download the ocp base images

```yml 
oc adm release mirror -a ${LOCAL_SECRET_JSON} --to-dir=${REMOVABLE_MEDIA_PATH}/mirror quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}
```

### Disk to Mirror OCP Base Images 

```yml
oc image mirror -a ${REG_CREDS} --from-dir=${REMOVABLE_MEDIA_PATH}/mirror "file://openshift/release:${OCP_RELEASE}*" ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}
```

### Extract openshift-install binary 

```yml 
oc adm release extract -a ${LOCAL_SECRET_JSON} --command=openshift-install "${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}"
```

### To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

```yml
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: icsp
spec:
  repositoryDigestMirrors:
  - mirrors:
    - kvmhost-raven.blackbird.com:8443/openshift4/ocp-release
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - kvmhost-raven.blackbird.com:8443/openshift4/ocp-release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```    
    

### List all available packages in a catalog 

```yml
oc-mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.18
```

### Create isc.yaml

```yml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  platform:
    channels:
    - name: stable-4.18
      minVersion: '4.18.12'
      maxVersion: '4.18.12'
      shortestPath: true
      type: ocp
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.18
      packages:
        - name: lvms-operator
          channels:
            - name: stable-4.18
        - name: local-storage-operator
          channels:
            - name: stable
        - name: loki-operator 
          channels: 
            - name: stable-6.2             
        - name: cluster-logging
          channels:
            - name: stable-6.2             
        - name: elasticsearch-operator
          channels:
            - name: stable-5.8
        - name: kubevirt-hyperconverged
          channels:
            - name: stable
        - name: kubernetes-nmstate-operator
          channels: 
            - name: stable                                  
        - name: mtv-operator
          channels: 
            - name: release-v2.8
        - name: metallb-operator
          channels: 
            - name: stable
```


### Mirror to Disk the operator images      
    
```yml
oc-mirror -c ./isc.yaml file:///root/quay-ops/oc-mirror/mirror1 --v2
```

### Disk to Mirror the operator images

```yml
oc-mirror -c ./isc.yaml --from file:///root/quay-ops/oc-mirror/mirror1 docker://kvmhost-raven.blackbird.com:8443 --v2
```

### KUBEMACPOOL VM ==================
```yml
oc annotate --overwrite -n openshift-cnv hco kubevirt-hyperconverged 'networkaddonsconfigs.kubevirt.io/jsonpatch=[{"op": "replace","path": "/spec/kubeMacPool","value": null}]'
```

#### FIX INGRESS ROUTE for Quay Application Deployment (push/pull)

```yml
oc extract secret/router-ca -n openshift-ingress-operator --to /tmp/tls.crt --confirm 
oc create cm custom-ca --from-file=ca-bundle.crt=/tmp/tls.crt -n openshift-config 
oc patch proxy/cluster --type=merge --patch='{"name":{"trustedCA":{"name":"custom-ca"}}}'
```

### RedHat Registry Login  

```yml 
oc registry login --registry="quay-enterprise.apps.cluster-5hxx5.dynamic.redhatworkshops.io" --auth-basic="quayadmin:XXXXXX" --to=pull-secret
```


### Update RedHat Pull Secret in OpenShift Cluster.  

```yml 
oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' >pull-secret
podman login -u quayadmin -p Asimov@123 quay-enterprise.apps.cluster-5hxx5.dynamic.redhatworkshops.io --authfile pull-secret
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=pull-secret
```

### Approval CSR OpenShift 
```yml 
oc get csr -o name | xargs oc adm certificate approve 
```

### Update HTPasswd secret. 
```yml 
oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd --dry-run=client -o yaml -n openshift-config | oc replace -f -
```


### TLS certificate unknown authority login
```yml 
oc rsh -n openshift-authentication <oauth-openshift-pod> cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > ingress-ca.crt
```

### OCP Etcd Backup using NFS. Add privileged SCC. 
```yml
oc adm policy add-scc-to-user privileged -z openshift-backup -n ocp-etcd-backup 
```

### Configure Ingress Router Workload. 
```yml
oc edit ingresscontroller default -n openshift-ingress-operator

spec: 
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
```


### DEFAULT ROUTE URL 
```yml
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')

oc extract secret/$(oc get ingresscontroller -n openshift-ingress-operator default -o json | jq '.spec.defaultCertificate.name // "router-certs-default"' -r) -n openshift-ingress --confirm

sudo mv tls.crt /etc/pki/ca-trust/source/anchors/

sudo update-ca-trust enable

sudo podman login -u kubeadmin -p $(oc whoami -t) $HOST
```

### Drain node forcefully 

```yml
oc adm drain node/<node-name> --ignore-daemonsets=true --delete-emptydir-data=true --disable-eviction=true
oc delete node node-name 
reboot -> worker node 
```


### If Facing Control-plane-machine-set failed replicas. 

```yml
oc get co | grep "control-plane-machine-set"
control-plane-machine-set                  4.14.33   False       False         True       12h     Missing 3 available replica(s)

oc delete controlplanemachineset.machine.openshift.io cluster -n openshift-machine-api
controlplanemachineset.machine.openshift.io "cluster" deleted

```
### control-plane-machine-set operator degraded in UPI install Azure/AWS 
https://access.redhat.com/solutions/7021642


# OLM Disable
https://docs.redhat.com/en/documentation/openshift_container_platform/4.9/html/operators/administrator-tasks#olm-restricted-networks-operatorhub_olm-restricted-networks

```yml
oc patch OperatorHub cluster --type json     -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
```

### RedHat Container Registries failed to communicate with Catalog index. 

```yml
curl -s -o /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-isv https://security.access.redhat.com/data/55A34A82.txt

cat /etc/containers/policy.json
{
  "default": [
      {
          "type": "insecureAcceptAnything"
      }
  ],
  "transports":
    {
      "docker-daemon":
          {
              "": [{"type":"insecureAcceptAnything"}]
          },
      "docker":
        {
          "registry.redhat.io/redhat/certified-operator-index": [
            {
              "type": "signedBy",
              "keyType": "GPGKeys",
              "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-isv"
            }
          ],
          "registry.redhat.io/redhat/community-operator-index": [
            {
              "type": "signedBy",
              "keyType": "GPGKeys",
              "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-isv"
            }
          ],
          "registry.redhat.io/redhat/redhat-marketplace-index": [
            {
              "type": "signedBy",
              "keyType": "GPGKeys",
              "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-isv"
            }
          ],
          "registry.redhat.io": [
            {
              "type": "signedBy",
              "keyType": "GPGKeys",
              "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release"
            }
          ]
        }
    }
}

podman pull registry.redhat.io/redhat/certified-operator-index:v4.14 --log-level debug --tls-verify=false
Copying blob 1be5daf91edf done
Copying blob 4b15e491ebc4 done
Copying blob 1cafa764d625 done

[corehub@drprsscvmbst1 ~]$ podman images
REPOSITORY                                                                            TAG                   IMAGE ID      CREATED       SIZE
registry.redhat.io/redhat/certified-operator-index                                    v4.14                 a901f20bf876  7 hours ago   1.45 GB
```

### Create RedHat catalogsource file. 

apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhat-operator-index
  namespace: openshift-marketplace 
spec:
  sourceType: grpc
  grpcPodConfig:
      priorityClassName: system-cluster-critical
      securityContextConfig: restricted
  image: quay-registry.apps.hub2.dr1.ocp.prs/openshiftlocal/ redhat/redhat-operator-index:v4.14 
  displayName: redhat operators
  publisher: RedHat 
  updateStrategy:
    registryPoll: 
      interval: 30m

cd /var/lib/kubelet/pki 
remove the certificate and reboot the node. if csr ceritificate is not coming. 


### VSSC Cluser is disconnected, Disable auto download of Boot Sources.
```yml
oc patch hco kubevirt-hyperconverged -n openshift-cnv \
  --type json -p '[{"op": "replace", "path": \
  "/spec/featureGates/enableCommonBootImageImport", \
  "value": false}]'
```

### Create a NodeNetworkConfigurationPolicy object to map the OVN-Kubernetes secondary network to an Open vSwitch (OVS) bridge:
```yml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: localnet-nncp
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ''
  desiredState:
    ovn:
      bridge-mappings:
      - localnet: localnet-network
        bridge: br-ex
        state: present
```
### Create a NetworkAttachmentDefinition object
```yml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: localnet-nad
  namespace: default
spec:
  config: |-
    {
            "cniVersion": "0.3.1",
            "name": "localnet-network",
            "type": "ovn-k8s-cni-overlay",
            "topology": "localnet",
    }
```

### Apply the manifest
```yml
oc create -f localnet-nncp.yaml -f localnet-nad.yaml
```


### Upgrade : ocp4.12 to 4.12.x latest 
```yml
oc adm release mirror --from=quay.io/openshift-release-dev/ocp-release:4.18.1-x86_64 --to=mreg01f2cr22u06.cerapreprod.mnr.har:8443/openshift4/ocp-release --to-release-image=mreg01f2cr22u06.cerapreprod.mnr.har:8443/openshift4/ocp-release:4.18.1

oc create secret generic my-pull-secret --from-file=.dockerconfigjson=<path-to-pull-secret> --type=kubernetes.io/dockerconfigjson
oc secrets link default my-pull-secret --for=pull

oc adm upgrade --to-image=mreg01f2cr22u06.cerapreprod.mnr.har:8443/openshift4/ocp-release:4.12.74 --allow-explicit-upgrade --force
```

### Change MTU
```yaml
nmcli connection modify "ens1f0np0" 802-3-ethernet.mtu 9100
nmcli connection modify "ens2f0np0" 802-3-ethernet.mtu 9100
nmcli connection modify "bond0" 802-3-ethernet.mtu 9100
nmcli connection modify "br-kvm" 802-3-ethernet.mtu 9100
nmcli connection modify "vlan-176" 802-3-ethernet.mtu 9100

nmcli connection modify "ens1f1np1" 802-3-ethernet.mtu 9000
nmcli connection modify "ens2f1np1" 802-3-ethernet.mtu 9000
nmcli connection modify "vlan-602" 802-3-ethernet.mtu 9000
nmcli connection modify "bond1" 802-3-ethernet.mtu 9000
```


###  Uninstall ODF Node
```yml
oc adm drain node/<node-name> --ignore-daemonsets=true --delete-emptydir-data=true --disable-eviction=true

oc get storagecluster -n openshift-storage

oc -n openshift-storage rsh deploy/rook-ceph-tools ceph osd tree
```

### Enable Ceph Tool Pod 
```yml
oc patch OCSInitialization ocsinit -n openshift-storage --type json --patch  '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'

blockdev --flushbufs /dev/mapper/mpathc
multipath -f mpathc

for a in sd{b..i} ; do  wipefs -af /dev/$a ; done
```



openshift-install agent wait-for bootstrap-complete --log-level=debug

openshift-install agent wait-for install-complete --log-level=debug          


### AD Integration with OCP 4.
Create a Secret object that contains the bindPassword field:
```yml
$ oc create secret generic ldap-secret --from-literal=bindPassword=####@### -n openshift-config

$ cat ad-sync-user.yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  annotations:
    include.release.openshift.io/ibm-cloud-managed: "true"
    include.release.openshift.io/self-managed-high-availability: "true"
    release.openshift.io/create-only: "true"
  name: cluster
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: htpass-secret
    mappingMethod: claim
    name: AD_USERS
    type: HTPasswd
  - ldap:
      attributes:
        email:
        - mail
        id:
        - sAMAccountName
        name:
        - cn
        preferredUsername:
        - sAMAccountName
      bindDN: CN=ocp bind,OU=RHEV_VM,DC=epfindia,DC=gov,DC=in
      bindPassword:
        name: ldap-bind-password-2bgmf
      insecure: true
      url: ldap://192.168.1.135:389/OU=RHEV_VM,DC=epfindia,DC=gov,DC=in?sAMAccountName
    mappingMethod: claim
    name: ldap
    type: LDAP
```
or 
```yml
$ oc create secret generic ldap-secret --from-literal=bindPassword=1nfs@Depl0y@2024 -n openshift-config

$ cat cris-ad-sync-user.yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  annotations:
    include.release.openshift.io/ibm-cloud-managed: "true"
    include.release.openshift.io/self-managed-high-availability: "true"
    release.openshift.io/create-only: "true"
  name: cluster
spec:
  identityProviders:
  - ldap:
      attributes:
        email:
        - mail
        id:
        - dn
        name:
        - cn
        preferredUsername:
        - uid
      bindDN: uid=aposdeploy,cn=users,cn=accounts,dc=authz,dc=prs
      bindPassword:
        name: ldap-secret
      insecure: true
      url: ldap://drprssclbidam01.authz.prs/cn=accounts,dc=authz,dc=prs?uid
    mappingMethod: claim
    name: CRIS_LDAP
    type: LDAP
```