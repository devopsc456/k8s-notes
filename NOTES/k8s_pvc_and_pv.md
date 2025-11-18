# K8S Internal Volume Lifecycle

- We can maintain internal volume lifecycle using **PVC** and **PV**.

## Why we maintain internal volume lifecycle?

- In a Kubernetes cluster we can't guarantee when a pod will go down or which node it will come up on again.
- We connect all nodes to **NFS (Network File System)** and maintaining this becomes difficult.

---

## PVC (Persistent Volume Claim) Object

- After creating the Pod in the Kubernetes cluster, we request a **PVC** object for storage.
- PVC is a **namespace-level** object.
- PVC is **bound** to PV.

---

## PV (Persistent Volume)

- PV is a **cluster-level** object.
- There can be multiple PV objects present in the Kubernetes cluster.
- PV is connected to the **NFS server**.

---

### Key Points

- One PVC is **bound** to one PV object (**one-to-one relationship**).
- Pods **cannot directly connect** to PV objects; only **PVC** connects to the pod.
- PV object requests resources (memory or storage) from the NFS server.


````md
# Example YAML for PVC and PV

## MongoDB Pod YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: mongodb
  namespace: prod
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongocon
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: devdb
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: devdb@123
        volumeMounts:
        - name: mongonfsvol
          mountPath: /data/db
      volumes:
      - name: mongonfsvol
        persistentVolumeClaim:
          claimName: mongodb-pvc  
---
apiVersion: v1
kind: Service
metadata:
  name: mongosvc
  namespace: prod
spec:
  type: ClusterIP
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
`---

- In the above MongoDB pod creation, we created the pod along with the PVC, but since the PVC was not available initially, the MongoDB pod went into a **Pending** state.

- We can describe the pod to check why it is in a pending state using the following command:

```bash
kubectl describe <pod_name> -n <namespace>
````

**Error Message:**

```
Warning  FailedScheduling  43s   default-scheduler  0/3 nodes are available: persistentvolumeclaim "mongodb-pvc" not found. preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.
```

# Usecase - 1

## PersistentVolumeClaim YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  namespace: prod # This is correct
spec:
  accessModes:
    - ReadWriteMany  # Must match the PV's access mode
  resources:
    requests:
      storage: 1Gi  # Must match with PV's size or be less than the PV's size
````

---

* `mongodb` specified `volumeClaim` name must match with the PVC YAML specified name, then only MongoDB pod can connect with PVC object.
* After creating the PVC object, the PVC goes into **Pending** state because the corresponding PV object is not present at cluster level. Also, no **StorageClass** is available in the background to dynamically provision a PV.

---

### Check PVC Status Command

```bash
kubectl describe pvc <pvc_name> -n <namespace>
```

---

### Error Message

```
Normal  FailedBinding  10s (x7 over 93s)  persistentvolume-controller  no persistent volumes available for this claim and no storage class is set
```

# Usecase - 2

- First bring up the **MongoDB pod**
- After that bring up the **PV (PersistentVolume)** → Here PV becomes available
- After that bring up the **PVC (PersistentVolumeClaim)**

---

If **PV** is available, then **PVC** will automatically get **Bound** to the available PV.

# Pictorial Representation for PVC and PV

 <img width="1600" height="816" alt="Screenshot 2025-11-18 103022" src="https://github.com/user-attachments/assets/4c1efd23-8eab-4b98-bb8a-0a2dead1aa88" />

