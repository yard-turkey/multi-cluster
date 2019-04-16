## Pattern 1

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


2. Deploy your application
