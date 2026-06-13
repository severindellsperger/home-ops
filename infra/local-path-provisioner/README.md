Currently, we are using the local-path-provisioner to provide a default StorageClass for our Kubernetes cluster.
This allows us to create PersistentVolumeClaims (PVCs) that will automatically be bound to local storage on the nodes.
Installation is done via the following command:

```bash
kubectl apply -f infra/local-path-provisioner/app/app/local-path-provisioner.yaml
```

Also, it needs:
```bash
kubectl label namespace local-path-storage pod-security.kubernetes.io/enforce=privileged
```

```
#### Add Default Storage Class
```
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: default-storage-class
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
```