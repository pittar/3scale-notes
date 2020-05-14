# Azure File - RWX for 3Scale

Azure File can be used for `RWX` storage in a Kubernetes cluster, but it has some limitations due to the fact that it is `cifs` based.  There is also at least one open issue with Microsoft that prevents permissions to be properly set in the mounted volume.  Until this is fixed, the following work around is required.

Most of the instructions for setting up Azure File dynamic provisioning for OpenShift can be found here: [Azrue File Dynamic Provisioning](https://docs.openshift.com/container-platform/4.4/storage/dynamic-provisioning.html#azure-file-definition_dynamic-provisioning).  The only difference is there are extra "parameters" to add to the *StorageClass* definition.  Those are explained below.

## Find the UID of your 3Scale Project

First, you will need to find the *UID* of your 3Scale project.  If you haven't created a 3Scale project in OpenShift, make sure you do that first!

To find the UID, run the following command:

```
$ oc get namespace 3scale -o yaml

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

In the example above, the `UID` is the the first part of the `openshift.io/sa.scc.uid-range` value.  For the namespace above, that value would be `1000600000`.

Make note of the UID, you will need it soon.

## Enable Azure File in OpenShift

You will need a new `ClusterRole` and a `StorageClass`.

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

There are sample yaml files in the [resources](resources) directory of this repo.


Now, apply the files in order and bind the *ClusterRole* to the `system:serviceaccount:kube-system:persistent-volume-binder` user:

```
$ oc apply -f azure-file-clusterrole.yaml

$ oc adm policy add-cluster-role-to-user system:azure-cloud-provider system:serviceaccount:kube-system:persistent-volume-binder

$ oc apply -f azure-file-storageclass.yaml
```

This StorageClass will be *specifically* for the 3Scale RWX persistent volume.  Of course, you can make anther Azure File StorageClass for generic use if you like.

You're now ready to install 3Scale!
