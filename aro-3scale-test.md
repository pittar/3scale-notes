# 3Scale 2.8 on ARO with Azure File for RWX Volume

## Step 1: Basic Setup

First, create a 3Scale project and apply your Red Hat pull secret.  You can generate a [service account pull secret here](https://access.redhat.com/terms-based-registry/).

```
oc new-project 3scale
oc apply -f <pull secret>.yaml -n 3scale
```

## Step 2: Create Two New Azure File Storage Classes

Next, we'll make two new storage classes based on Azure File.  A "standard" Azure File storage class, and one specifically for the 3Scale project.

First, find the UID of the 3Scale project:

```
oc get namespace 3scale -o yaml

apiVersion: v1
kind: Namespace
metadata:
  annotations:
    openshift.io/description: ""
    openshift.io/display-name: ""
    openshift.io/requester: kube:admin
    openshift.io/sa.scc.mcs: s0:c25,c0
    openshift.io/sa.scc.supplemental-groups: 1000600000/10000
    openshift.io/sa.scc.uid-range: 1000600000/10000
  creationTimestamp: "2020-05-14T17:35:14Z"
  name: 3scale
  resourceVersion: "123025"
  selfLink: /api/v1/namespaces/3scale
  uid: 192f4868-9e44-40ac-aba5-3abe5e1b6ab8
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
```

You will need a new `ClusterRole` and two `StorageClass` resources.

**azure-file-clusterrole.yaml**
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:azure-cloud-provider
rules:
- apiGroups: ['']
  resources: ['secrets']
  verbs:     ['get','create']
```

**3scale-azure-file-storageclass.yaml**
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata: 
  name: 3scale-azure-file
provisioner: kubernetes.io/azure-file
mountOptions: 
- uid=<your 3Scale project UID here>
- gid=0
- mfsymlinks
- cache=strict
parameters: 
  skuName: Standard_LRS
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

**standard-azure-file-storageclass.yaml**
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata: 
  name: standard-azure-file
provisioner: kubernetes.io/azure-file
parameters: 
  skuName: Standard_LRS
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

Now, apply the files in order and bind the *ClusterRole* to the `system:serviceaccount:kube-system:persistent-volume-binder` user:

```
$ oc apply -f azure-file-clusterrole.yaml

$ oc adm policy add-cluster-role-to-user system:azure-cloud-provider system:serviceaccount:kube-system:persistent-volume-binder

$ oc apply -f 3scale-azure-file-storageclass.yaml

$ oc apply -f standard-azure-file-storageclass.yaml
```

You will now have two new RWX storage classes available:  `3scale-azure-file` and `standard-azure-file`

## Install the 3Scale Operator

From the OpenShift Admin console, navigate to "Operators -> Operator Hub" and select the "Red Hat Integration - 3Scale" operator.
Install the latest version (threescale-2.8) into the "**3Scale**" project by clicking "Subscribe".

## Install a 3Scale API Manager with "Standard" Azure File

To see if a normal Azure File PVC will work, try to install an API Manager into the 3Scale project using the the `standard-azure-file` storage class.

First, take note of your cluster router domain.  It will start witih "apps".  For example, `apps.cfs05m1p.eastus.aroapp.io`.  You will need this for the wildcard domain of your APIManager custom resource:

**api-manager.yaml**
```
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: 3scale
  namespace: 3scale
spec:
  wildcardDomain: <your openshift router domain>
  resourceRequirementsEnabled: false
  system:
    fileStorage:
      persistentVolumeClaim:
        storageClassName: standard-azure-file
```

Apply this resource to your cluster and wait for 3Scale to install.  It will take a few minutes.

```
oc apply -f api-manager.yaml -n 3scale
```

You may see pods failing and re-starting during the install. This is normal, so be patient.
