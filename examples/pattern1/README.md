## Pattern 1

### Prereqs

Make sure you have a federated cluster setup running in OCP 4.0. See [guide](https://docs.google.com/document/d/1QvSvA2JxSvqRFjc92gqFZnn9-aPd-bA6RcErsqbbW58/edit#).

### Summary

In this pattern, we will simply deploy a Federated application across our clusters using the *EXISTING* default storageclass as scratch space and persistence.

### Steps
1. Create a [pvc](https://github.com/yard-turkey/multi-cluster/edit/master/examples/pattern1/pvc-default.yaml) on each cluster where you will deploy your application.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: ebs-default
spec:
  storageClassName: gp2
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
*NOTE* This is not federated, guessing it will be at somepoint?

```
# oc --context=cluster1 create -f pvc-default.yaml 
persistentvolumeclaim/ebs-default created

# oc --context=cluster2 create -f pvc-default.yaml 
persistentvolumeclaim/ebs-default created
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
