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
1. Create a [federated pvc](https://github.com/yard-turkey/multi-cluster/edit/master/examples/pattern1/pvc-federated.yaml) to create storage using the default storageclass which your applicaton will consume.

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

2. Deploy a simple [busybox](https://github.com/yard-turkey/multi-cluster/edit/master/examples/pattern1/busybox-deployment.yaml) application

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
