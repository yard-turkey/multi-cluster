## Pattern 2 - Specific StorageClass Federated

### Prereqs

Make sure you have a federated cluster setup running in OCP 4.0. See [guide](https://docs.openshift.com/container-platform/4.1/welcome/index.html?extIdCarryOver=true&sc_cid=701f2000001Css0AAC).

### Summary

In this pattern, we will deploy a Federated application across our clusters federating any resources needed (StorageClass, PVC, Application, Secrets, etc...). We will model the following scenarios:

- Pattern 2 A: Simple Federated App using Dynamic Storage StorageClass
- Pattern 2 B: Install Rook-Ceph

### Pattern 2 - A: Simple Federated App using Dynamic StorageClass

#### Steps
1. Enable the PersistentVolumeClaims API Resource type for Federation.

```
# kubefed2 enable PersistentVolumes --federation-namespace=federation-test and --registry-namespace=federation-test
# kubefed2 enable PersistentVolumeClaims --federation-namespace=federation-test and --registry-namespace=federation-test
```
