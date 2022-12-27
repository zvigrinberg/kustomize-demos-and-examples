# kustomize-demos-and-examples
Provide some useful kustomize features that can be used across use-cases.

## Remove namespace field from Openshift resources using OpenAPI json definitions and Delete SMP on a path in the resource manifest
1. Show that Route and deployment config files are with metadata.namespace field
```shell
[zgrinber@zgrinber kustomize-demos-and-example]$ cat kustomize-crd-delete-namespace/base/deployment-config.yaml kustomize-crd-delete-namespace/base/route.yaml 
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: dc-test
  namespace: myNamespace
spec:
  replicas: 1
  selector:
    deployment-config.name: dc-test
  strategy:
    resources: {}
  template:
    metadata:
      labels:
        deployment-config.name: dc-test
    spec:
      containers:
      - image: busybox
        name: default-container
        command: ["sleep"]
        args:
          - "infinity"
        resources: {}
        envFrom:
          - configMapRef:
              name: dc-config
  test: false
  triggers:
    - type: ConfigChange
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    component: apiserver
    provider: kubernetes
  name: kubernetes
  namespace: my-namespace
spec:
  port:
    targetPort: https
  to:
    kind: "Service"
    name: kubernetes
    weight: 100
```
2. Now Build the kustomization target with kustomize, and see that namespaces fields disappeared from route and deploymentconfig manifests:
```shell
 kustomize build kustomize-crd-delete-namespace
 # or using directly kubectl or oc binaries
 oc kustomize kustomize-crd-delete-namespace
 
 apiVersion: v1
data:
  hello: world
  important: value
  newone: "yes"
  oldone: "no"
  password: changeme
  test: debug
  user: john
kind: ConfigMap
metadata:
  name: dc-config-dhmcm95h8f
---
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: dc-test
spec:
  replicas: 1
  selector:
    deployment-config.name: dc-test
  strategy:
    resources: {}
  template:
    metadata:
      labels:
        deployment-config.name: dc-test
    spec:
      containers:
      - args:
        - infinity
        command:
        - sleep
        envFrom:
        - configMapRef:
            name: dc-config-dhmcm95h8f
        image: busybox
        name: default-container
        resources: {}
  test: false
  triggers:
  - type: ConfigChange
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    component: apiserver
    provider: kubernetes
  name: kubernetes
spec:
  port:
    targetPort: https
  to:
    kind: Service
    name: kubernetes
    weight: 100
```

## Remove Resources completely using Delete SMP and annotationSelector
1. Print Kustomization file to the see the included resources, and print rolebinding.yaml, to see its content, and that it's annotated with saasiRemoveResource: true
```shell
zgrinber@zgrinber kustomize-demos-and-example]$ cat kustomize-smp-delete-resources/base/kustomization.yaml kustomize-smp-delete-resources/base/rolebinding.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: build

resources:
- deployment.yaml
- rolebinding-owner.yaml
- rolebinding.yaml 
- first-configmap.yaml
- second-configmap.yaml
- service-account.yaml


patches:
  - target:
      annotationSelector: "saasiRemoveResource=true"

    patch: |-
     apiVersion: someVersion
     kind: whateverKind
     metadata:
       name: whateverName 
     $patch: delete
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
    openshift.io/description: Allows deploymentconfigs in this namespace to rollout
      pods in this namespace.  It is auto-managed by a controller; remove subjects
      to disable.
    saasiRemoveResource: true
  creationTimestamp: "2022-11-19T15:13:18Z"
  name: system:deployers
  namespace: gitlab-system
  resourceVersion: "164854425"
  uid: ff2dea7c-ddc1-4419-89c2-92c3df8dff71
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:deployer
subjects:
- kind: ServiceAccount
  name: deployer
  namespace: gitlab-system
```

2. Now Build the kustomization target with kustomize and see that rolebinding.yaml is excluded from kustomize build' output 
```shell
[zgrinber@zgrinber kustomize-demos-and-example]$ kustomize build kustomize-smp-delete-resources
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: null
  labels:
    layer: smp-delete-resource-demo
  name: my-sa
---
apiVersion: v1
data:
  key1: value1
  key2: value2
kind: ConfigMap
metadata:
  annotations:
    saasiRemoveResource: "false"
  labels:
    layer: smp-delete-resource-demo
  name: first-configmap
---
apiVersion: v1
data:
  key3: value3
  key4: value4
kind: ConfigMap
metadata:
  labels:
    layer: smp-delete-resource-demo
  name: second-configmap
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app
    layer: smp-delete-resource-demo
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
      layer: smp-delete-resource-demo
  strategy: {}
  template:
    metadata:
      labels:
        app: app
        layer: smp-delete-resource-demo
    spec:
      containers:
      - command:
        - bash
        - -c
        - sleep infinity
        envFrom:
        - configMapRef:
            name: first-configmap
        - configMapRef:
            name: second-configmap
        image: ubi8/ubi:8.5-226
        name: ubi
```
