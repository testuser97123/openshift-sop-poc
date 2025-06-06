```yml
[root@bastion ~]# vim config.yaml
DEFAULT_TAG_EXPIRATION: 2w
FEATURE_USER_INITIALIZE: true
FEATURE_USER_CREATION: true
BROWSER_API_CALLS_XHR_ONLY: false
FEATURE_QUOTA_MANAGEMENT: true
FEATURE_BUILD_SUPPORT: true
FEATURE_PROXY_CACHE: true
FEATURE_PROXY_STORAGE: true
FEATURE_DIRECT_LOGIN: true
REGISTRY_TITLE: Red Hat Quay
REGISTRY_TITLE_SHORT: Red Hat Quay
TESTING: false
FEATURE_UI_V2: true
FEATURE_UI_V2_REPO_SETTINGS: true
FEATURE_AUTO_PRUNE: true
ROBOTS_DISALLOW: false
CREATE_NAMESPACE_ON_PUSH: true
SETUP_COMPLETE: true
CREATE_PRIVATE_REPO_ON_PUSH: false
TAG_EXPIRATION_OPTIONS:
- 2w
TEAM_RESYNC_STALE_TIME: 60m
DISTRIBUTED_STORAGE_CONFIG:
    default:
        - RHOCSStorage
        - access_key: wOpKg0giEZo1IEukfiJB
          bucket_name: quay-bucket-eb66eda9-dbca-4e60-b1b9-653592713622
          hostname: s3-openshift-storage.apps.cluster-xxxxx.dynamic.redhatworkshops.io
          is_secure: true
          port: "443"
          secret_key: 6qYhSMB3hVc7ncQ1whRZFYG/OeeQ1TYYPWFz+WbM
          storage_path: /datastorage/registry
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS: []
DISTRIBUTED_STORAGE_PREFERENCE:
  - default
SERVER_HOSTNAME: quay-enterprise.apps.cluster-xxxxx.dynamic.redhatworkshops.io
SUPER_USERS:
  - quayadmin

[root@bastion ~]# vim ssl.cert 
-----BEGIN CERTIFICATE-----
xxx
-----END CERTIFICATE-----

[root@bastion ~]# oc create secret generic quay-secret --from-file=config.yaml=./config.yaml --from-file=ssl.cert=./ssl.cert 

[root@bastion ~]# vim quay-registry.yaml 
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: quay-enterprise
  namespace: quay-registry
spec:
  components:
    - kind: clair
      managed: true
    - kind: postgres
      managed: true
    - kind: objectstorage
      managed: false
    - kind: redis
      managed: true
    - kind: horizontalpodautoscaler
      managed: true
    - kind: route
      managed: true
    - kind: mirror
      managed: true
    - kind: monitoring
      managed: true
    - kind: tls
      managed: true
    - kind: quay
      managed: true
    - kind: clairpostgres
      managed: true
  configBundleSecret: quay-secret

[root@bastion ~]# oc create -f quay-registry.yaml 

[root@bastion ~]# oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' >pull-secret

[root@bastion ~]# podman login -u quayadmin -p Asimov@123 quay-enterprise.apps.cluster-xxxxx.dynamic.redhatworkshops.io --authfile pull-secret

[root@bastion ~]# oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=pull-secret

[root@bastion ~]# vim deploy.yaml
apiVersion: v1
items:
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: sise
    name: sise
    namespace: default
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: sise
    strategy:
      rollingUpdate:
        maxSurge: 25%
        maxUnavailable: 25%
      type: RollingUpdate
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: sise
      spec:
        containers:
        - image: quay-enterprise.apps.cluster-xxxxx.dynamic.redhatworkshops.io/local/sise
          imagePullPolicy: Always
          name: sise
          ports:
          - containerPort: 9876
            protocol: TCP

[root@bastion ~]# oc create -f deploy.yaml

[root@bastion ~]# oc get po
NAME                   READY   STATUS    RESTARTS   AGE
sise-8d5b45777-mqbv6   1/1     Running   0          114s
sise-8d5b45777-r6246   1/1     Running   0          116s

[root@bastion ~]# oc get po -o wide
NAME                   READY   STATUS    RESTARTS   AGE    IP            NODE                     NOMINATED NODE   READINESS GATES
sise-8d5b45777-mqbv6   1/1     Running   0          119s   10.132.2.52   worker-cluster-xxxxx-3   <none>           <none>
sise-8d5b45777-r6246   1/1     Running   0          2m1s   10.135.0.43   worker-cluster-xxxxx-2   <none>           <none>

[root@bastion ~]# oc describe po sise-8d5b45777-mqbv6 | grep -i image
    Image:          quay-enterprise.apps.cluster-xxxxx.dynamic.redhatworkshops.io/local/sise
    Image ID:       quay-enterprise.apps.cluster-xxxxx.dynamic.redhatworkshops.io/local/sise@sha256:fb60a895123154f7a9438f0953db7c3e917eb011947708afc0957426b6d85b43
  Normal  Pulling         2m18s  kubelet            Pulling image "quay-enterprise.apps.cluster-xxxxx.dynamic.redhatworkshops.io/local/sise"
  Normal  Pulled          2m17s  kubelet            Successfully pulled image "quay-enterprise.apps.cluster-xxxxx.dynamic.redhatworkshops.io/local/sise" in 369ms (369ms including waiting)

[root@bastion ~]# oc exec -i sise-8d5b45777-mqbv6 -t -- curl  http://localhost:9876/info
{"host": "localhost:9876", "version": "0.5.0", "from": "127.0.0.1"}

```