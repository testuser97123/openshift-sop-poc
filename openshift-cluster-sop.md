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