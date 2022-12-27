# Enable auto configuration change on deployment config Openshift.

## Background
As you know , Kubernetes Still doesn't support monitoring change of configmap when it changed to trigger a rolling update of deployments that are using    it, so you need to rollout deployment yourself 

## Goal: To automatically Create ConfigMap on each configuration change(environment variable change,added or remove) and patch the deployment config that use that ConfigMap with new configmap name in order to trigger a rolling update of pods.
         
### Way to acheive that:
 
 - I'll be Using kustomize ConfigMapGenerator to create configmaps with hash suffix(hash based on the content and is altered every time config map content is changing),the values will be configMapGenerator' literals for this demo(in real use case you can refer to a file that can be modified earlier,maybe in ci/cd pipeline earlier to kustomize build or by some other means).
 - kustomize' support for crds will be used and CRDS openapi json definition for openshift resources like deploymentconfig will be used(are in this repo), otherwise kustomize will not patch accordingly openshift resources.
 
 ### Procedure
 
 We'll start with the following configmap data(located in [here](./base/kustomization.yaml))
 ```yaml
 apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment-config.yaml
crds:
  - crds-openshift/openshift-deploymentconfig.json
  - crds-openshift/openshift-route.json
configMapGenerator:
  - name: dc-config
    literals:
      - hello=world
      - user=john
      - password=changeme
      - newone=yes
      - oldone=no
      - important=value
 ```
 
 1. from repository' root, Run the following command to deploy a test deploymentconfig:
 ```shell
 kustomize build . | oc apply -f -
 ```
 ```shell
 configmap/dc-config-b8k7gfbh7k created
 deploymentconfig.apps.openshift.io/dc-test created
 ```
 
 2. See that a pod created and running:
 ```shell
oc get pods
NAME               READY   STATUS      RESTARTS   AGE
dc-test-1-deploy   0/1     Completed   0          21s
dc-test-1-l94sx    1/1     Running     0          18s
```
3. Now show the configmap data:
```shell
oc describe cm dc-config-b8k7gfbh7k

Name:         dc-config-b8k7gfbh7k
Namespace:    test-kustomize
Labels:       <none>
Annotations:  <none>

Data
====
password:
----
changeme
user:
----
john
hello:
----
world
important:
----
value
newone:
----
yes
oldone:
----
no

```
 4. Verify that the container of pod contains these environment variables:
 ```shell
  oc exec -it dc-test-1-l94sx -- env | grep -E ^[a-z]
  hello=world
  important=value
  newone=yes
  oldone=no
  password=changeme
  user=john
 ```
 5. Now add new environment variable test=debug in kustomization.yaml file here [here](./base/kustomization.yaml).
 6. now run once again kustomize build and apply manifests to cluster:
 ```shell
 kustomize build . | oc apply -f -
 configmap/dc-config-dhmcm95h8f created
 deploymentconfig.apps.openshift.io/dc-test configured
 ```
 7. see that this rolled out a new deployment and it got the new configmap
 ```shell
  oc get pods
NAME               READY   STATUS      RESTARTS   AGE
dc-test-1-deploy   0/1     Completed   0          14m
dc-test-2-deploy   0/1     Completed   0          23s
dc-test-2-dwtxj    1/1     Running     0          20s

oc describe pod dc-test-2-dwtxj | grep -i environment -A2
  Environment Variables from:
    dc-config-dhmcm95h8f  ConfigMap  Optional: false


 ```
 8. You can see that configmap created and pod got updated with it,now let's verify that new value added to configmap and to container pod:
  ```shell
  oc describe configmap/dc-config-dhmcm95h8f
  Name:         dc-config-dhmcm95h8f
  Namespace:    test-kustomize
  Labels:       <none>
  Annotations:  <none>

  Data
  ====
  test:
  ----
  debug
  user:
  ----
  john
  hello:
  ----
  world
  important:
  ----
  value
  newone:
  ----
  yes
  oldone:
  ----
  no
  password:
  ----
  changeme

  ```
  9. Validate the new environment variable is inside pod' container:
   ```shell
    oc exec -it dc-test-2-dwtxj -- env | grep -E ^[a-z]
    test=debug
    user=john
    hello=world
    important=value
    newone=yes
    oldone=no
    password=changeme

   ```
