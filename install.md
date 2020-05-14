# Installing 3Scale

3Scale is installed with the Oprator from the internal OpenShift OperatorHub.

## 3Scale Version

These notes are for **3Scale 2.8**.

# Prerequisites

## RWX/S3

For the install of an actual `APIManager` instance, you will also have to have `RWX` or `S3` storage available.  Options for this include **OpenShift Container Storage**, **S3 storage**, or **Azure File** to name a few.

## Pull Secret

You will need a access token for the Red Hat registry.  To get one:
1. Login to the [Red Hat registry to create a service account and download a token](https://access.redhat.com/terms-based-registry)
2. Once you have created a service account, download the "OpenShift Secret" as a yaml file.  You will need this later.


## OpenShift Container Storage

If you have *OpenShift Container Storage* available in your cluster, then you already have access to an `RWX` storage class.  This is the option you should choose for both simplicity and stability.

## S3 Storage - AWS

If you are on AWS, you can setup an S3 bucket to use for the the shared system storage.  This is [documented in the 3Scale 2.8 documentation](https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.8/html/installing_3scale/install-threescale-on-openshift-guide#amazon_simple_storage_service_3scale_emphasis_filestorage_emphasis_installation)

## S3 Storage - Noobaa

If you don't have access to an `RWX` storage class or an AWS S3 bucket, you can install [Noobaa](https://www.noobaa.io/) or another S3-compliant storage provider in your cluster.  For instructions on using Noobaa for S3 storage, please [see the instructions in this 3scale-noobaa repository](https://github.com/pittar/3scale-noobaa).

## Azure File (RWX)

Azure File can be used as an RWX storage class, however, it is CIFS based and a bug with setting permissions.  Microsoft is aware and are working on the issue.  Until then, there is a [resonably easy work around](azurefile.md).

# Installing 3Scale

## Create your 3Scale Project

First, you will need an OpenShift project.  You can call this whatever you like, but for the purpose of this doc we will call it `3scale`.  If you choose something else, make sure you update your yaml files accordingly!

```
$ oc new-project 3scale
```

## Add your Registry Token

Earlier you downloaded a registry token in the form of an OpenShift Secret (yaml file).  Apply this to your project now:

```
$ oc apply -f <your token name>.yaml -n 3scale
```

## Prepare System Storage

Unless you already have an `RWX` StorageClass available to you, you will have to follow the instructions above to prepare an S3 bucket and associated Secret, or a special [Azure File storage class specific to your 3Scale project](azurefile.md).

## Install the 3Scale Operator

In the OpenShift Console in the *Administrator* view:
* Select *Operators -> OperatorHub* from the left navigation panel 
* Find the *Red Hat Integration - 3Scale* Operator.  Do **not** chose the *Community* version.
* Click *Install*.  This will bring you to the *Create Operator Subscription* page.
* Select the following options:
    * Installation Mode: A specific namespace on the cluster
    * Installed namespace: 3scale (or whatever name you chose for your project)
    * Update channel: threescale-2.8
    * Approval strategy: Manual
    * Click **Subscribe**

This will take you to the list of "Installed Operators".  3Scale should be in an *Upgrade Pending* state.
* Click on "3scale-operator"
* Click on "1 requires approval"
* Click "Preview Install Plan"
* Click "Approve"

This will install the 3Scale Operator.  This will also ensure that 3Scale will not upgrade (e.g. from 2.8 to 2.9) without your approval.

You can now switch back to the "Developer" view and click on "Topology" to see the 3Scale operator has installed.

## Create an APIManager

For a basic install of 3Scale, you can use the following two options; one for `RWX` StorageClass, and one for `S3`.

**apimanager.yaml - Example if using RWX storage:**
```
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: 3scale
  namespace: 3scale
spec:
  wildcardDomain: apps.ocp.azure.pitt.ca
  resourceRequirementsEnabled: false
  system:
    fileStorage:
      persistentVolumeClaim:
        storageClassName: 3scale-azure-file
```

**apimanager.yaml - Example if using S3 storage:**
```
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: 3scale
  namespace: 3scale
spec:
  wildcardDomain: apps.ocp.azure.pitt.ca
  resourceRequirementsEnabled: false
  system:
    fileStorage:
      simpleStorageService:
        configurationSecretRef:
          name: s3-auth

```

In the **S3** example, `s3-auth` is the name of the secret that contains the S3 credentials and info [as described in the 3Scale docs](https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.8/html/installing_3scale/install-threescale-on-openshift-guide#amazon_simple_storage_service_3scale_emphasis_filestorage_emphasis_installation).

Finally, apply the yaml in your namespace:
```
$ oc apply -f apimanager.yaml -n 3scale
```

You should see 3Scale start to deploy.  It will take a few minutes for all images to pull and pods to come online.

Once everything is running, you can login to your 3Scale master at:
`https://master.<wildcard domain from apimanager.yaml>`

You can find the master username and password in the `Secret` in your 3scale project named `system-seed`.