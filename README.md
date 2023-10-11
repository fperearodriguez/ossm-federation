# OSSM federated in OpenShift

The objective of this repository is to setup a federated service mesh by using the OpenShift Service Mesh [federation module.](https://docs.openshift.com/container-platform/4.13/service_mesh/v2x/ossm-federation.html)

## Prerequisites
- Two OCP cluster installed. In this laboratory, two SNO clusters have been installed in AWS.
- OCP 4.13 version
- OSSM 2.4.X
- In this laboratory, _istio-system_ is used as control plane's namespace.
- Cluster names: _cluster-1_ & _cluster-2_.
- Two applications used: [sleep](./3-rbac/deploy.yaml) and _helloworld_. In _cluster-1_, the helloworld application used is this [deployment](./2-ossm-apps/cluster-1/deploy-helloworld.yaml), and in _cluster-2_ the application used is [this one.](./2-ossm-apps/cluster-2/deploy-helloworld.yaml)

## Top-level diagram
How-to: Export service from _cluster-2_ cluster and import it into _cluster-1_ cluster.

<img src=./0-images/exportandimport.png width=700>


## OSSM resources

1. Create SMCP & SMMR
2. Fetch Istio CAcert from each Service Mesh:

Cluster-1 cluster:
```bash
oc get configmap istio-ca-root-cert -o jsonpath='{.data.root-cert\.pem}' > 1-ossm-resources/1-federation/remote-cluster-1-mesh-cert.pem
```

Cluster-2 cluster:
```bash
oc get configmap istio-ca-root-cert -o jsonpath='{.data.root-cert\.pem}' > 1-ossm-resources/1-federation/remote-cluster-2-mesh-cert.pem
```

3. Create the cluster-1 configmap in the cluster-2 cluster:

Cluster-2 cluster:
```bash
oc -n istio-system create configmap cluster-1-ca-root-cert --from-file=root-cert.pem=./1-ossm-resources/1-federation/remote-cluster-1-mesh-cert.pem
```

Cluster-1 cluster:
```bash
oc -n istio-system create configmap cluster-2-ca-root-cert --from-file=root-cert.pem=./1-ossm-resources/1-federation/remote-cluster-2-mesh-cert.pem
```

4. Retrieve AWS LB ip addresses:

Cluster-1 cluster:
```bash
AWS_LB_SM_EAST=$(oc get svc cluster-2-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' -n istio-system)
```

Cluster-2 cluster:
```bash
AWS_LB_SM_WEST=$(oc get svc cluster-1-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' -n istio-system)
```

5. Create the ServiceMeshPeer resources in both cluster:

Cluster-1 cluster:
```bash
oc apply -f 1-ossm-resources/1-federation/servicemeshpeer-cluster-1.yaml
```

Cluster-2 cluster:
```bash
oc apply -f 1-ossm-resources/1-federation/servicemeshpeer-cluster-2.yaml
```

6. Create exportedServiceSet in the _cluster-2_ cluster:

```bash
oc apply -f 1-ossm-resources/1-federation/exportedserviceset-cluster-2.yaml
```

```bash
  status:
    exportedServices:
    - exportedName: helloworld-canary.my-awesome-project.svc.cluster-1-exports.local
      localService:
        hostname: helloworld-canary.my-awesome-project.svc.cluster.local
        name: helloworld-canary
        namespace: my-awesome-project
    - exportedName: sleep.my-awesome-project.svc.cluster-1-exports.local
      localService:
        hostname: sleep.my-awesome-project.svc.cluster.local
        name: sleep
        namespace: my-awesome-project
```

7. Create importedServiceSet in the _cluster-1_ cluster:

```bash
oc apply -f 1-ossm-resources/1-federation/importedserviceset-cluster-1.yaml
```

Once the _importedServiceSet_ is created, it may take some minutes to reconcile and discover the new services.

```bash
 status:
    importedServices:
    - exportedName: helloworld-canary.my-awesome-project.svc.cluster-1-exports.local
      localService:
        hostname: helloworld-canary.my-awesome-project.svc.cluster.local
        name: helloworld-canary
        namespace: my-awesome-project
    - exportedName: sleep.my-awesome-project.svc.cluster-1-exports.local
      localService:
        hostname: sleep.my-awesome-project.svc.cluster-2-imports.local
        name: sleep
        namespace: my-awesome-project
```