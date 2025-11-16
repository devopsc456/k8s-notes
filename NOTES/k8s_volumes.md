# Example without k8s Volumes

## spring-app.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springapp
  namespace: prod 
spec:
  replicas: 2
  selector: 
    matchLabels:
      app: springapp
  template:
    metadata:
      labels:
        app: springapp
    spec:  
      containers:
      - name: springappcon
        image: kkeducation12345/spring-app:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: MONGO_DB_HOSTNAME
          value: mongosvc
        - name: MONGO_DB_USERNAME
          value: devdb
        - name: MONGO_DB_PASSWORD
          value: devdb@123
---
apiVersion: v1
kind: Service
metadata:
  name: springappsvc
  namespace: prod
spec:
  type: NodePort
  selector:
    app: springapp
  ports:
  - port: 80
    targetPort: 8080
````

## mongodb-without-volumes.yaml

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata: 
  name: mongodb
  namespace: prod
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongocon
        image: mongo:8.0.9-noble
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: devdb
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: devdb@123
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
```
Here is your text formatted as **Markdown (.md)**:

# MongoDB Pod Behavior Without Persistent Storage

- MongoDB was created and scheduled on **node 184**.
- If the MongoDB pod gets deleted for any reason, Kubernetes will automatically create a new pod because it was created using a **ReplicaSet**.
- **Problem:** Since the pod has **no persistent volume**, the data stored inside the original pod will not be available after it is deleted.
- The new MongoDB pod may be scheduled on a **different node**, and Kubernetes does **not guarantee** that it will be created on the same node again.

## Example Scenario
- Original MongoDB pod scheduled on **node 184**
- Pod deleted → ReplicaSet creates a **new pod**
- New MongoDB pod scheduled on **node 196**
- Data stored in the original pod is **lost** because it was not persisted externally

**Conclusion:**  
Without volumes or persistent storage, **MongoDB data will be lost** whenever a pod restarts, gets rescheduled, or moves to a different node.

Here is your provided text converted into **Markdown (.md)** format:

````md
# Example with HostPath Volume

## mongodb-hostpath-vol.yaml

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata: 
  name: mongodb
  namespace: prod
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongocon
        image: mongo:8.0.9-noble
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: devdb
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: devdb@123
        volumeMounts:
        - name: mongovol
          mountPath: /data/db
      volumes:
      - name: mongovol
        hostPath:
          path: /mongobkp
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
````

### Notes

* Before HostPath volumes, `emptyDir` was used, but **it is deprecated**.
* Using HostPath volumes allows data to be backed up, **but data inconsistency may occur**.

---

## How Data Becomes Inconsistent

1. MongoDB pod is created first on **node 184** and data backup is taken there.
2. Due to some issue, the MongoDB pod is deleted on node 184.
3. Kubernetes schedules a **new MongoDB pod on a different node (e.g., node 196)**.
4. Now backup is taken from node 196, **not from node 184**, causing **data inconsistency**, not total data loss.

---

### Conclusion

* **HostPath volumes are not recommended** for production use because of **node dependency and data inconsistency issues**.

Here is your content properly formatted in **Markdown (.md)**:

# NFS Volumes in Kubernetes

## Setup the NFS Server

**Port used by NFS:** `2049`

### Step-by-step

#### **Step 1:** Launch an EC2 machine and connect to it

#### **Step 2:** Update package manager
```bash
sudo apt update -y
````

#### **Step 3:** Allow inbound traffic on **port 2049**

#### **Step 4:** Install NFS server software

```bash
sudo apt install nfs-kernel-server -y
```

#### **Step 5:** Create NFS shared directory

```bash
sudo mkdir -p /mnt/nfs_share
sudo chown nobody:nogroup -R /mnt/nfs_share/   # Grant write permission to all clients
sudo chmod 777 -R /mnt/nfs_share/
```

#### **Step 6:** Configure NFS export rules

```bash
sudo vi /etc/exports
```

Add the following line:

```
/mnt/nfs_share/ *(rw,sync,no_subtree_check,no_root_squash)
```

#### **Step 7:** Export and restart NFS service

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

#### **Step 8:** Verify NFS service is running

```bash
ps -ef | grep -i "nfs"
```

---

## Configuring the NFS Client (All Kubernetes Nodes)

```bash
sudo apt update -y
sudo apt install nfs-common -y
```

---

## Example: Using NFS Volume in Kubernetes

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
        image: mongo:8.0.9-noble
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
        nfs:
         server: 172.31.6.199 (perferrable use NFS private IP)
         path: /mnt/nfs_share
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
```

---

## Notes

* Using **NFS (Network File System)** prevents data loss because data is stored on a **separate NFS server** accessible by all cluster nodes.
* **Limitation:** If the NFS server crashes, **data loss may occur**.
  → To prevent this, take **snapshots or backups** of the NFS server.

---

## Useful Kubernetes Commands

### Force delete a ReplicaSet (or any resource)

```bash
kubectl delete replicaset.apps/springapp-7c57dbf6 --force --grace-period=0
```

### Access a running pod shell

```bash
kubectl exec -it mongodb-ncmzz -n prod -- bash
```
