# 3Scale 2.8 on ARO with Azure File for RWX Volume

## Step 1: Basic Setup

First, create a 3Scale project and apply your Red Hat pull secret.  You can generate a [service account pull secret here](https://access.redhat.com/terms-based-registry/).

```
oc new-project 3scale
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

Once 3Scale is fully initialized, take a look at the `system-sidkiq` pod logs.  They will likely end in a error like this:
```
Exception -- {:exception=>{:class=>ActiveRecord::RecordNotFound, :message=>"Couldn't find Account with 'id'=4 [WHERE `accounts`.`provider` = ?]", :backtrace=>["/opt/system/vendor/bundle/ruby/2.5.0/gems/activerecord-5.0.7.2/lib/active_record/relation/finder_methods.rb:353:in `raise_record_not_found_exception!'", "/opt/system/vendor/bundle/ruby/2.5.0/gems/activerecord-5.0.7.2/lib/active_record/relation/finder_methods.rb:479:in `find_one'", "/opt/system/vendor/bundle/ruby/2.5.0/gems/activerecord-5.0.7.2/lib/active_record/relation/finder_methods.rb:458:in `find_with_ids'", "/opt/system/vendor/bundle/ruby/2.5.0/gems/activerecord-5.0.7.2/lib/active_record/relation/finder_methods.rb:66:in `find'"]}, :parameters=>{:context=>"Job raised exception", :job=>{"class"=>"SignupWorker::ImportSimpleLayoutWorker", "args"=>[4], "retry"=>true, "queue"=>"critical", "jid"=>"6911169869dc607f6a9ecf1a", "created_at"=>1595377612.8962731, "bid"=>"dl1uJu1W4zRGbw", "enqueued_at"=>1595377612.8963265}, :jobstr=>"{\"class\":\"SignupWorker::ImportSimpleLayoutWorker\",\"args\":[4],\"retry\":true,\"queue\":\"critical\",\"jid\":\"6911169869dc607f6a9ecf1a\",\"created_at\":1595377612.8962731,\"bid\":\"dl1uJu1W4zRGbw\",\"enqueued_at\":1595377612.8963265}"}}
```

Once 3Scale is installed, view the `system-seed` secret and take note of the "master" password (not token).  Click on `system-app` route and you will be taken to the "master" admin console.  Login with user "master" and the password you copied from the `system-seed` password.

Once logged in, click on "Account" near the top of the screen, then "Create" near the right side of the screen.  Create a new user and password, then use "test" as the org/group name.  This will be your new 3Scale tenant.

Click on the "Admin Domain" link (test-admin.<openshift router url>) to be taken to your new tenant in a new browser tab. 

First, change back to your "master" account tab and click the "users" link near the top of the screen.  Then, click the green "Activate" link beside your new account.

Now, back to the tab with your new tenant and login with the username and password you just created.

Once you have logged in, from the top nav bar, click "Dashboard" then select "Audience".

From the left-nav, click "Developer Portal -> Visit Portal"

You will be taken to a new tab with a 3Scale "Not Found" page.  This is due to the Azure File RWX volume.

In the `system-sidekiq` pod logs, you will again see the error:
```
Exception -- {:exception=>{:class=>Errno::EPERM, :message=>"Operation not permitted @ apply2files - /opt/system/public//system/pitt/2020/07/22/desk-d665dcff745a7ff5.jpg", :backtrace=>["/opt/rh/rh-ruby25/root/usr/share/ruby/fileutils.rb:1243:in `chmod'", "/opt/rh/rh-ruby25/root/usr/share/ruby/fileutils.rb:1243:in `chmod'", "/opt/rh/rh-ruby25/root/usr/share/ruby/fileutils.rb:921:in `block in chmod'", "/opt/rh/rh-ruby25/root/usr/share/ruby/fileutils.rb:920:in `each'"]}, :parameters=>{:context=>"Job raised exception", :job=>{"class"=>"SignupWorker::ImportSimpleLayoutWorker", "args"=>[4], "retry"=>true, "queue"=>"critical", "jid"=>"6911169869dc607f6a9ecf1a", "created_at"=>1595377612.8962731, "bid"=>"dl1uJu1W4zRGbw", "enqueued_at"=>1595378203.327179, "error_message"=>"Operation not permitted @ apply2files - /opt/system/public//system/pitt/2020/07/22/desk-34a4272f8cec16c2.jpg", "error_class"=>"Errno::EPERM", "failed_at"=>1595377612.910722, "retry_count"=>4, "retried_at"=>1595377862.2894173}, :jobstr=>"{\"class\":\"SignupWorker::ImportSimpleLayoutWorker\",\"args\":[4],\"retry\":true,\"queue\":\"critical\",\"jid\":\"6911169869dc607f6a9ecf1a\",\"created_at\":1595377612.8962731,\"bid\":\"dl1uJu1W4zRGbw\",\"enqueued_at\":1595378203.327179,\"error_message\":\"Operation not permitted @ apply2files - /opt/system/public//system/pitt/2020/07/22/desk-34a4272f8cec16c2.jpg\",\"error_class\":\"Errno::EPERM\",\"failed_at\":1595377612.910722,\"retry_count\":4,\"retried_at\":1595377862.2894173}"}}
```

When I terminal into the pod and run `mount | grep cifs`, the result is:
```
$ mount | grep cifs
//f5a333f166ef14640a1afb9.file.core.windows.net/cluster-rm8v6-dynamic-pvc-15292cad-83e4-4022-bc12-a75e04ba02f3 on /opt/system/public/system type cifs (rw,relatime,vers=3.0,cache=strict,username=f5a333f166ef14640a1afb9,uid=0,noforceuid,gid=1000600000,forcegid,addr=52.239.153.40,file_mode=0777,dir_mode=0777,soft,persistenthandles,nounix,serverino,mapposix,rsize=1048576,wsize=1048576,bsize=1048576,echo_interval=60,actimeo=1)
```

Notice the UID is 0.  The gid for some reason has the UID value.

### Delete 3Scale

Now, delete this 3Scale API Manager instance to try again with the Azure File StorageClass that is forced to use the correct UID.

```
oc delete apimanager 3scale -n 3scale
```

It should only take a moment to delete all the pods.

## Install a 3Scale API Manager with "3scale" Azure File StorageClass

Update your `APIManager` custom resource to use the `3scale-auzure-file` storag class.

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
        storageClassName: 3scale-azure-file
```

Apply this resource to your cluster.
```
oc apply -f api-manager.yaml -n 3scale
```

Follow the same steps as above to login to the master admin portal, create a new tenant, and test the Developer Portal.

This time, it will work!

## Bottom Line

For some reason, a normal Azure File PVC gets mounted with a UID of 0 and a GID that is supposed to be the UID.
