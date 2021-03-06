## Pattern 1

### Prereqs

Make sure you have a federated cluster setup running in OCP 4.0. See [guide](https://docs.google.com/document/d/1QvSvA2JxSvqRFjc92gqFZnn9-aPd-bA6RcErsqbbW58/edit#).

### Summary

In this pattern, we will deploy a Federated application across our clusters federating any resources needed (StorageClass, PVC, Application, Secrets, etc...). We will model the following scenarios:

- Pattern 1 A: Simple Federated App using Default StorageClass
- Pattern 1 B: Federate Object Bucket and Object Bucket Claim with AWS s3 Provisioning StorageClass
- Pattern 1 C: Federate Object Consuming App to Connect to Existing AWS S3 Bucket

### Pattern 1 - A: Simple Federated App using Default StorageClass

#### Steps
1. Enable the PersistentVolumeClaims API Resource type for Federation.

```
# kubefed2 enable PersistentVolumes --federation-namespace=federation-test and --registry-namespace=federation-test
# kubefed2 enable PersistentVolumeClaims --federation-namespace=federation-test and --registry-namespace=federation-test
```

2. Create a [federated pvc](https://github.com/yard-turkey/multi-cluster/edit/master/examples/pattern1/pvc-federated.yaml) to create storage using the default storageclass which your applicaton will consume.

```yaml
apiVersion: types.federation.k8s.io/v1alpha1
kind: FederatedPersistentVolumeClaim
metadata:
  name: ebs-default
spec:
  template:
    metadata:
      name: ebs-default
    spec:
      storageClassName: gp2
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
  placement:
    clusterNames:
    - cluster2
    - cluster1
```
Now check and see that our PVC's were created and bound

```
# oc get federatedpersistentvolumeclaims
NAME           AGE
ebs-default   12m

[root@ip-10-0-30-112 ~]# oc --context=cluster2 get pvc
NAME           STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ebs-default    Bound     pvc-255ca901-606f-11e9-ab6d-0e86289ef8a8   1Gi        RWO            gp2            61m

[root@ip-10-0-30-112 ~]# oc --context=cluster1 get pvc
NAME           STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ebs-default    Bound     pvc-1fa6ca91-606f-11e9-8b5f-0e8757c373c8   1Gi        RWO            gp2            62m

```

3. Deploy a simple [busybox](https://github.com/yard-turkey/multi-cluster/edit/master/examples/pattern1/busybox-deployment.yaml) application

```
# oc --context=cluster1 create -f busybox-deployment.yaml 
federateddeployment.types.federation.k8s.io/busybox created

# oc --context=cluster2 get pods
NAME                       READY   STATUS    RESTARTS   AGE
busybox-6666ffbc4b-x9hfr   1/1     Running   0          36s

# oc --context=cluster1 get pods
NAME                                             READY   STATUS    RESTARTS   AGE
busybox-6666ffbc4b-zwlb5                         1/1     Running   0          42s
federation-controller-manager-78fdf7f5c4-9nrmn   1/1     Running   0          3h34m
```

### Pattern 1 B: Federate Object Bucket and Object Bucket Claim with AWS s3 Provisioning StorageClass

#### Steps

** Assumes federation is fully installed and running in your control plane and federation-test namespace exists on both clusters

1. Enable the core federation resources.

```
# kubefed2 enable Namespaces --federation-namespace=federation-test and --registry-namespace=federation-test
# kubefed2 enable deployments.apps --federation-namespace=federation-test and --registry-namespace=federation-test
# kubefed2 enable deployments.extensions --federation-namespace=federation-test and --registry-namespace=federation-test
# kubefed2 enable Secrets --federation-namespace=federation-test and --registry-namespace=federation-test
# kubefed2 enable CustomResourceDefinitions --federation-namespace=federation-test and --registry-namespace=federation-test
# kubefed2 enable StorageClasses --federation-namespace=federation-test and --registry-namespace=federation-test
```
single line command
```
kubefed2 enable Namespaces --federation-namespace=federation-test and --registry-namespace=federation-test;kubefed2 enable deployments.extensions --federation-namespace=federation-test and --registry-namespace=federation-test;kubefed2 enable CustomResourceDefinitions --federation-namespace=federation-test and --registry-namespace=federation-test;kubefed2 enable StorageClasses --federation-namespace=federation-test and --registry-namespace=federation-test
```

2. Create the FederatedNamespace for federation-test

```yaml
apiVersion: types.federation.k8s.io/v1alpha1
kind: FederatedNamespace
metadata:
  name: federation-test
  namespace: federation-test
spec:
  placement:
    clusterNames:
    - cluster2
    - cluster1
```

execute the namespace yaml
```
# oc create -f federated-ns.yaml
federatednamespace.types.federation.k8s.io/federation-test created

# oc get federatednamespaces
NAME              AGE
federation-test   7s

```

3. Manually create the [OB/OBC CRD](https://github.com/yard-turkey/lib-bucket-provisioner/blob/master/deploy/customResourceDefinitions.yaml) on each of your clusters (THIS IS NOT PROPAGATED VIA FEDERATION YET).

```
Once this is properly propagated via federation - this step will no longer be needed, but for now you need to do this step on each cluster AND as good practice also do step 4 below to add the crd into the federated resources just so federation tracks and knows about the crd.
```


4. Create the Federated [OB CRD](https://github.com/yard-turkey/multi-cluster/edit/master/examples/pattern1/ob-crd-federated.yaml) for Bucket provisioning

```yaml
apiVersion: types.federation.k8s.io/v1alpha1
kind: FederatedCustomResourceDefinition
metadata:
  name: objectbuckets.objectbucket.io
  namespace: federation-test
spec:
  template:
    group: objectbucket.io
    versions:
      - name: v1alpha1
        served: true
        storage: true
    names:
      kind: ObjectBucket
      listKind: ObjectBucketList
      plural: objectbuckets
      singular: objectbucket
      shortNames:
        - ob
        - obs
    scope: Cluster
    subresources:
      status: {}
  placement:
    clusterNames:
    - cluster2
    - cluster1
---
apiVersion: types.federation.k8s.io/v1alpha1
kind: FederatedCustomResourceDefinition
metadata:
  name: objectbucketclaims.objectbucket.io
  namespace: federation-test
spec:
  template:
    group: objectbucket.io
    versions:
      - name: v1alpha1
        served: true
        storage: true
    names:
      kind: ObjectBucketClaim
      listKind: ObjectBucketClaimList
      plural: objectbucketclaims
      singular: objectbucketclaim
      shortNames:
        - obc
        - obcs
    scope: Namespaced
    subresources:
      status: {}
    placement:
      clusterNames:
      - cluster2
      - cluster1
```
Create the CRD
```
# oc create -f ob-crd-federated.yaml 
federatedcustomresourcedefinition.types.federation.k8s.io/objectbuckets.objectbucket.io created
federatedcustomresourcedefinition.types.federation.k8s.io/objectbucketclaims.objectbucket.io created

# oc get federatedcustomresourcedefinitions
NAME                                 AGE
objectbucketclaims.objectbucket.io   3m6s
objectbuckets.objectbucket.io        3m6s
```
*NOTE:* This is not working, it does not create the CRD on each cluster - for now we will manually create the CRD for
        OB and OBC on each cluster as a workaround.

```
# oc --context=cluster1 create -f crd.yaml 
customresourcedefinition.apiextensions.k8s.io/objectbuckets.objectbucket.io created
customresourcedefinition.apiextensions.k8s.io/objectbucketclaims.objectbucket.io created

# oc --context=cluster2 create -f crd.yaml 
customresourcedefinition.apiextensions.k8s.io/objectbuckets.objectbucket.io created
customresourcedefinition.apiextensions.k8s.io/objectbucketclaims.objectbucket.io created
```

5. Enable the OB and OBC API Resource types for this Federation example.

```
# kubefed2 enable ObjectBuckets --federation-namespace=federation-test and --registry-namespace=federation-test
# kubefed2 enable ObjectBucketClaims --federation-namespace=federation-test and --registry-namespace=federation-test
```

6. Create the Federated Owner Reference [Secret](https://github.com/yard-turkey/multi-cluster/edit/master/examples/pattern1/owner-secret-federated.yaml) for Bucket Provisioning in AWS and execute it on your primary cluster.

```yaml
apiVersion: types.federation.k8s.io/v1alpha1
kind: FederatedSecret
metadata:
  name: s3-bucket-owner
  namespace: federation-test
spec:
  template:
    data:
      AWS_ACCESS_KEY_ID: <base64 encoded aws access key>
      AWS_SECRET_ACCESS_KEY: <base64 encoded aws secret key>
    type: Opaque
  placement:
    clusterNames:
    - cluster2
    - cluster1
```

Create the secret and verify.

```
# oc create -f owner-secret-federated.yaml

# oc --context=cluster2 get secrets -n federation-test
NAME                                TYPE                                  DATA   AGE
builder-dockercfg-2x6rf             kubernetes.io/dockercfg               1      40h
builder-token-29njj                 kubernetes.io/service-account-token   4      40h
builder-token-8m4r9                 kubernetes.io/service-account-token   4      40h
cluster2-cluster1-dockercfg-nr4xd   kubernetes.io/dockercfg               1      22h
cluster2-cluster1-token-bgj8n       kubernetes.io/service-account-token   4      22h
cluster2-cluster1-token-kc42m       kubernetes.io/service-account-token   4      22h
default-dockercfg-77wz7             kubernetes.io/dockercfg               1      40h
default-token-2vn5k                 kubernetes.io/service-account-token   4      40h
default-token-lbpln                 kubernetes.io/service-account-token   4      40h
deployer-dockercfg-js659            kubernetes.io/dockercfg               1      40h
deployer-token-g89m7                kubernetes.io/service-account-token   4      40h
deployer-token-mfnnz                kubernetes.io/service-account-token   4      40h
s3-bucket-owner                     Opaque                                2      61s <here it is>
```


7. Create the StorageClass to dynamically provision buckets

```yaml
apiVersion: types.federation.k8s.io/v1alpha1
kind: FederatedStorageClass
metadata:
  name: s3-buckets
  namespace: federation-test
spec:
  template:
    provisioner: aws-s3.io/bucket
    parameters:
      region: us-west-1
      secretName: s3-bucket-owner
      secretNamespace: s3-provisioner
    reclaimPolicy: Delete
  placement:
    clusterNames:
    - cluster1
    - cluster2
```

Create the StorageClass

```
# oc create storageclass-federated.yaml

# oc get sc

# oc --context=cluster2 get sc

```
*NOTE:* This is not working, it does not create the SC on each cluster - for now we will manually create as workaround.


8. Create our FederatedDeployment for the AWS-S3-Provisioner (when containerized - for now manually run).
```
# ./awss3provisioner-scott -master https://api.screeley-cluster1.screeley.sysdeseng.com -kubeconfig /root/.kube/config -alsologtostderr -v=2
```

9.  Enable the CRD api type.

```
# kubefed2 enable objectbucketclaims
customresourcedefinition.apiextensions.k8s.io/federatedobjectbucketclaims.types.federation.k8s.io created
federatedtypeconfig.core.federation.k8s.io/objectbucketclaims.objectbucket.io created in namespace federation-system

# kubefed2 enable objectbuckets
customresourcedefinition.apiextensions.k8s.io/federatedobjectbuckets.types.federation.k8s.io created
federatedtypeconfig.core.federation.k8s.io/objectbuckets.objectbucket.io created in namespace federation-system
```

10. (After above is working) Create our FederatedObjectBucketClaim on CLUSTER1 only (greenfield).

```yaml
apiVersion: types.federation.k8s.io/v1alpha1
kind: FederatedObjectBucketClaim
metadata:
  name: green-obc
  namespace: federation-test
spec:
  template:
    spec:
      bucketName: screeley-fed-bucket
      storageClassName: s3-buckets
  placement:
    clusterNames:
    - cluster1

```


11. Create another FederatedObjectBucketClaim on remaining clusters to connect to existing bucket created above.


