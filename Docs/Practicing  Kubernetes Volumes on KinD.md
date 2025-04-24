# Practicing Kubernetes Volumes on KinD & Simulating NFS

This comprehensive guide walks through how to use KinD (Kubernetes in Docker) to practice **all** core Kubernetes volume types, and provides multiple approaches for simulating network storage endpoints like NFS - both with dnsmasq and without it.

## Table of Contents

1. [Introduction]
2. [Built-in Ephemeral & Config Volumes]
3. [hostPath & Static PersistentVolume]
4. [Dynamic PVs with Local-Path-Provisioner]
5. [Network & CSI-Based Volumes]
6. [Simulating NFS with dnsmasq]
7. [Simulating NFS without dnsmasq]
8. [Advanced Configuration Options]
9. [Summary Table]
10. [Troubleshooting]

---

## 1. Introduction

KinD is a conformant Kubernetes distribution running inside Docker (or containerd) containers. You can mount host directories into KinD node containers, install CSI drivers, and run in-cluster provisioners—enabling you to experiment with virtually every volume type from ephemeral to cloud-backed.

This guide covers:

1. Built-in ephemeral & config-only volumes
    
2. `hostPath` & static `PersistentVolume`
    
3. Dynamic local PVs via Rancher's local-path-provisioner
    
4. Network & CSI-based volumes (NFS, iSCSI, Ceph)
    
5. Simulating NFS DNS names with dnsmasq
    
6. Alternative approaches without dnsmasq
    
7. Deploying an in-cluster NFS CSI provisioner
    
8. Host-mounted NFS server via extraMounts
    
9. A summary table with all options
    

---

## 2. Built-in Ephemeral & Config Volumes

These volume types require no special setup on KinD—they work out of the box:

- `emptyDir` - Temporary storage that exists for the pod's lifetime
- `configMap` - Mount configuration data as files
- `secret` - Mount sensitive data as files
- `downwardAPI` - Expose pod/container metadata to applications
- `projected` - Combine multiple volume sources into a single directory

```yaml
# Example: emptyDir
apiVersion: v1
kind: Pod
metadata:
  name: test-ephemeral
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","echo hello > /cache/data; sleep 3600"]
    volumeMounts:
    - mountPath: /cache
      name: cache-vol
  volumes:
  - name: cache-vol
    emptyDir: {}
```

All these volume types are excellent for testing application behavior with different configuration and data sources, without requiring any external dependencies.

---

## 3. `hostPath` & Static `PersistentVolume`

KinD nodes are Docker containers—so you can mount host directories into them to provide persistent storage.

1. Create a KinD config with `extraMounts`:
    
    ```yaml
    # kind-config.yaml
    apiVersion: kind.x-k8s.io/v1alpha4
    kind: Cluster
    nodes:
    - role: control-plane
      extraMounts:
      - hostPath: /home/alvin/data
        containerPath: /mnt/data
    ```
    
2. Create the cluster:
    
    ```bash
    kind create cluster --config kind-config.yaml
    ```
    
3. Define a static PV referencing that path:
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: static-pv
    spec:
      capacity:
        storage: 1Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: /mnt/data
    ```
    
4. Create a PVC to use this PV:
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: static-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      volumeName: static-pv  # Direct reference to the PV
      storageClassName: ""   # Empty string to indicate no dynamic provisioning
    ```
    

This approach gives you full control over where the data is stored on your host machine, making it easy to inspect, back up, or modify files directly.

---

## 4. Dynamic PVs with Local-Path-Provisioner

For a more production-like experience with automatic volume provisioning, you can install Rancher's CSI driver:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml
```

Now:

- `kubectl get sc` shows `local-path` as `Default`.
- Any `PersistentVolumeClaim` will provision under `/opt/local-path-provisioner` on nodes.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: local-path
```

To test this with a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-volume-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo Hello from dynamic PV > /data/test.txt; sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: data-vol
  volumes:
  - name: data-vol
    persistentVolumeClaim:
      claimName: dynamic-pvc
```

This approach mimics cloud provider storage classes that automatically provision volumes on demand.

---

## 5. Network & CSI-Based Volumes

For volumes like NFS, iSCSI, Ceph RBD, or cloud-provider disks:

1. **Deploy** or **stand up** the backing service (NFS server, Ceph cluster, etc.) accessible to KinD nodes.
    
2. **Install** the matching CSI driver (helm chart or YAML) in your cluster.
    
3. **Create** a `StorageClass` tied to that CSI driver.
    
4. **Provision** PVCs normally; pods mount them via the driver.
    

> **Note:** Cloud disks (AWS EBS, GCE PD, Azure Disk) require real cloud or a simulator—KinD cannot mock these by itself.

This approach allows you to experiment with more complex storage scenarios, such as:

- ReadWriteMany volumes shared across multiple pods
- Storage with specific performance characteristics
- External storage systems with their own snapshot, backup, and replication features

---

## 6. Simulating NFS Endpoints with dnsmasq

dnsmasq handles DNS and DHCP only—it **cannot** serve NFS protocols. However, you can:

- **Fake DNS names** for your NFS server (e.g., `nfs.k8s.local`).
- Still run an actual NFS server (in-cluster CSI or host-mounted).

### 6.1 Configure dnsmasq for KinD DNS

```bash
# Point nfs.k8s.local to KinD gateway (e.g., 172.18.0.1)
echo "address=/nfs.k8s.local/172.18.0.1" | sudo tee /etc/dnsmasq.d/kind-nfs.conf
sudo systemctl restart dnsmasq
# Flush DNS on Linux
sudo resolvectl flush-caches
```

### 6.2 Deploy In-Cluster NFS CSI Provisioner

This CSI driver uses a hostPath (or PVC) under the hood to serve NFS:

```bash
git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
kubectl apply -f nfs-subdir-external-provisioner/deploy/kubernetes/deployment.yaml
```

Create a `StorageClass` referencing `nfs.k8s.local` and the exported path:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-sc
provisioner: example.com/nfs
parameters:
  server: nfs.k8s.local
  path: /exports
reclaimPolicy: Delete
mountOptions:
  - vers=4.1
```

### 6.3 Host-Mounted NFS Server

1. **Install** `nfs-kernel-server` on your workstation:
    
    ```bash
    sudo apt install nfs-kernel-server
    echo "/home/alvin/exports *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
    sudo exportfs -rav
    ```
    
2. **Mount** `/home/alvin/exports` into each KinD node:
    
    ```yaml
    # kind-nfs-host.yaml
    apiVersion: kind.x-k8s.io/v1alpha4
    kind: Cluster
    nodes:
    - role: control-plane
      extraMounts:
      - hostPath: /home/alvin/exports
        containerPath: /exports
    ```
    
3. **Define** a PV for that NFS export:
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: host-nfs-pv
    spec:
      capacity:
        storage: 2Gi
      accessModes: [ReadWriteMany]
      nfs:
        server: nfs.k8s.local
        path: /exports
    ```
    

The advantage of using dnsmasq is that it allows you to use domain names that match your production environment, making the testing environment more realistic.

---

## 7. Simulating NFS without dnsmasq

If you prefer not to use dnsmasq, there are several alternative approaches:

### 7.1 Using IP Addresses Directly

1. **Find** your host machine's IP address on the Docker bridge network:
    
    ```bash
    # Get Docker bridge network IP (usually 172.17.0.1)
    HOST_IP=$(ip -4 addr show docker0 | grep -Po 'inet \K[\d.]+')
    echo $HOST_IP
    ```
    
2. **Install** NFS server on your host:
    
    ```bash
    sudo apt install nfs-kernel-server
    echo "/home/alvin/exports *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
    sudo exportfs -rav
    sudo systemctl restart nfs-kernel-server
    ```
    
3. **Create** a PV using the IP address:
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: host-nfs-pv-ip
    spec:
      capacity:
        storage: 2Gi
      accessModes: [ReadWriteMany]
      nfs:
        server: 172.17.0.1  # Replace with your actual host IP
        path: /home/alvin/exports
    ```
    

### 7.2 Using /etc/hosts in KinD Nodes

1. **Create** a KinD config that includes a setup script:
    
    ```yaml
    # kind-nfs-hosts-file.yaml
    apiVersion: kind.x-k8s.io/v1alpha4
    kind: Cluster
    nodes:
    - role: control-plane
      extraMounts:
      - hostPath: /home/alvin/exports
        containerPath: /exports
      # Script to add hosts entry
      extraPortMappings:
      - containerPort: 22
        hostPort: 2222
    ```
    
2. **Create** the cluster:
    
    ```bash
    kind create cluster --config kind-nfs-hosts-file.yaml
    ```
    
3. **Find** your host IP and add it to the KinD node's hosts file:
    
    ```bash
    HOST_IP=$(ip -4 addr show docker0 | grep -Po 'inet \K[\d.]+')
    docker exec -it kind-control-plane bash -c "echo \"$HOST_IP nfs.k8s.local\" >> /etc/hosts"
    ```
    
4. **Define** a PV as before:
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: host-nfs-pv
    spec:
      capacity:
        storage: 2Gi
      accessModes: [ReadWriteMany]
      nfs:
        server: nfs.k8s.local
        path: /exports
    ```
    

### 7.3 Running an In-Cluster NFS Server

For a completely self-contained solution:

1. **Deploy** an NFS server within the KinD cluster:
    
    ```yaml
    # nfs-server.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nfs-server
    spec:
      selector:
        matchLabels:
          app: nfs-server
      template:
        metadata:
          labels:
            app: nfs-server
        spec:
          containers:
          - name: nfs-server
            image: itsthenetwork/nfs-server-alpine:latest
            securityContext:
              privileged: true
            env:
            - name: SHARED_DIRECTORY
              value: /exports
            volumeMounts:
            - name: exports
              mountPath: /exports
          volumes:
          - name: exports
            emptyDir: {}
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: nfs-server
    spec:
      ports:
      - name: nfs
        port: 2049
      selector:
        app: nfs-server
    ```
    
2. **Deploy** the resources:
    
    ```bash
    kubectl apply -f nfs-server.yaml
    ```
    
3. **Create** a PV using the service:
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: in-cluster-nfs-pv
    spec:
      capacity:
        storage: 1Gi
      accessModes: [ReadWriteMany]
      nfs:
        server: nfs-server.default.svc.cluster.local
        path: /exports
    ```
    

This approach is entirely self-contained within the KinD cluster, with no external dependencies.

---

## 8. Advanced Configuration Options

### 8.1 Multi-Node Cluster with Storage Node

For more production-like setups:

```yaml
# kind-multi-node.yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
- role: worker
- role: worker
  # Dedicated storage node
- role: worker
  labels:
    storage: "true"
  extraMounts:
  - hostPath: /home/alvin/storage
    containerPath: /storage
```

Then create node affinity rules for storage:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

### 8.2 Testing Stateful Applications

For testing StatefulSets with persistent storage:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "local-path"
      resources:
        requests:
          storage: 100Mi
```

---

## 9. Summary Table

|Volume Category|KinD Setup|Provisioner / Driver|Use Case|
|---|---|---|---|
|Ephemeral & Config|None (builtin)|N/A|Temporary data, configuration|
|hostPath & Static PV|`extraMounts` in KinD config|hostPath|Testing with pre-defined data|
|Dynamic PVs (local-path)|Install local-path-provisioner CSI|`rancher.io/local-path`|Mimicking cloud dynamic provisioning|
|NFS with dnsmasq|dnsmasq + NFS server|Various NFS provisioners|Simulating enterprise NAS systems|
|NFS without dnsmasq|Direct IP or in-cluster NFS|Various NFS provisioners|Self-contained testing environment|
|Cloud Disks (EBS/GCE/Azure)|Requires real cloud or simulator|Official cloud CSI drivers|Testing cloud-native applications|
|Advanced Multi-Node|Multi-node KinD cluster with dedicated storage nodes|Various|Production-like topology testing|

---

## 10. Troubleshooting

### Common Issues and Solutions

1. **Volume mounting permission issues:**
    
    ```bash
    # Fix permissions on host directory
    sudo chmod -R 777 /home/alvin/exports
    ```
    
2. **NFS connections failing:**
    
    ```bash
    # Check if NFS server is running
    sudo systemctl status nfs-kernel-server
    
    # Verify exports
    showmount -e localhost
    
    # Check firewall rules
    sudo ufw status
    ```
    
3. **PVC stuck in pending:**
    
    ```bash
    # Check PVC status
    kubectl describe pvc <pvc-name>
    
    # Look for provisioner errors
    kubectl get events --sort-by='.lastTimestamp'
    ```
    
4. **KinD cannot access host services:**
    
    ```bash
    # Check Docker network setup
    docker network inspect kind
    
    # Verify connectivity from inside KinD
    kubectl run -it --rm debug --image=busybox -- ping <host-ip>
    ```
    

With this setup you can:

- Practice **all** Kubernetes volume types locally
- Choose between dnsmasq and simpler alternative approaches
- Combine in-cluster and host-mounted servers to cover NFS, Ceph, iSCSI, and more
- Build increasingly complex storage scenarios that mimic production environments

Happy testing!