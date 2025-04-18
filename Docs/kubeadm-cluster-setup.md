# **Step-by-Step Kubernetes Cluster Setup with `kubeadm` and Calico CNI (with Loopback IP Fix)**

---

## ⚙️ 1. **Prepare and Assign the API Server IP (Loopback Fix)**

If the `--apiserver-advertise-address` IP is **not assigned to any NIC**, add it to the loopback interface:


`sudo ip addr add 10.8.8.10/24 dev lo`

> ✅ This ensures the API server can bind to the advertised IP.

---

## 🔧 2. **Initialize the Kubernetes Control Plane**

```
sudo kubeadm init \   --pod-network-cidr=172.18.0.0/16 \   --apiserver-advertise-address=10.8.8.10
```



### 🔍 Flag Breakdown:

|Option|Purpose|
|---|---|
|`--pod-network-cidr=172.18.0.0/16`|CIDR block for pod IPs; must match Calico config|
|`--apiserver-advertise-address=10.8.8.10`|API server’s advertised IP address|

📌 Save the `kubeadm join` command printed at the end — you’ll use it to join worker nodes.

---

## 🧰 3. **Configure kubectl for the Admin User**

```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```


---

## 🌐 4. **Install the Calico CNI Plugin**

Calico enables pod-to-pod communication and supports network policy enforcement.

### ✅ Install Calico for the `172.18.0.0/16` pod network:

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```


> 💡 The default Calico manifest supports `--pod-network-cidr=192.168.0.0/16`.  
> If you're using `172.18.0.0/16`, no changes are needed for Calico v3.27+ — it auto-detects.

---

## ✅ 5. **Verify the Cluster is Ready**


`kubectl get nodes`

Expected output:

pgsql

CopyEdit

`NAME                STATUS   ROLES                  AGE     VERSION kube-control-plane  Ready    control-plane,master   5m      v1.32.3`

> If it's still `NotReady`, Calico might still be initializing — wait 30–60s.

---

## 🚀 6. **Join Worker Nodes**

SSH into each worker node and run the join command from Step 2. Example:

```
sudo kubeadm join 10.8.8.10:6443 \   --token <token> \   --discovery-token-ca-cert-hash sha256:<hash>
```


📌 If you lost the command:

```
kubeadm token create --print-join-command
```


---

## 👁️ 7. **Verify Node Join Status**

Back on the control plane:


`kubectl get nodes`

You should see:

pgsql

CopyEdit

`NAME                STATUS   ROLES                  AGE     VERSION kube-control-plane  Ready    control-plane,master   1h      v1.32.3 kube-worker-1       Ready    <none>                 10m     v1.32.3`

---

## 🧼 Optional Cleanup & Reset

To reinitialize the cluster:

```
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /etc/cni/net.d /var/lib/kubelet/pki
sudo systemctl restart containerd kubelet

```

---

## ✅ Summary of Key Steps

|Step|Description|
|---|---|
|`ip addr add`|Adds loopback IP for API server binding|
|`kubeadm init`|Bootstraps control plane|
|`kubectl setup`|Configures admin access|
|`kubectl apply -f calico.yaml`|Deploys Calico CNI|
|`kubeadm join`|Adds worker nodes|
|`kubectl get nodes`|Verifies all nodes are Ready|


---



## 🧠 Full Command:

bash

CopyEdit

`sudo kubeadm init --pod-network-cidr 172.18.0.0/16 \   --apiserver-advertise-address 10.8.8.10`

---

## 🔍 Detailed Breakdown

### 🔹 `sudo`

Runs the command with **root privileges**, which is required because:

- `kubeadm` interacts with low-level system components
    
- It writes configs to `/etc/kubernetes`
    
- It pulls and manages container images
    
- It configures systemd services like `kubelet`
    

---

### 🔹 `kubeadm init`

This is the core command that:

- **Bootstraps a Kubernetes control plane node**
    
- Initializes the cluster by:
    
    - Generating TLS certificates for secure communication
        
    - Deploying control plane components as **static pods**
        
    - Creating configuration files like `admin.conf`
        
    - Starting `etcd`, the `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler`
        

Think of this as the Kubernetes "master setup wizard."

---

### 🔹 `--pod-network-cidr=172.18.0.0/16`

> 🧱 **Defines the CIDR block Kubernetes will use for assigning IP addresses to Pods**

- This **must match your CNI (Container Network Interface) plugin**
    
- For example, Cilium, Flannel, or Calico expect this to be **predefined** so they can hook into the pod network
    
- It **should not overlap** with:
    
    - Your VM or cloud subnet
        
    - The host’s IP range
        
    - Service CIDRs
        

> In your case:

bash

CopyEdit

`172.18.0.0/16`

This allocates ~65,000 pod IPs (for large clusters or labs). You’ll typically see one `/24` block (e.g. `172.18.1.0/24`) assigned per node.

---

### 🔹 `--apiserver-advertise-address=10.8.8.10`

> 🌐 **Specifies the IP address that the Kubernetes API server will bind to and advertise**

- This is the **node's private IP address** — the one **other nodes (workers)** will use to **communicate with the control plane**
    
- It should be:
    
    - A reachable IP from all worker nodes
        
    - A stable IP of your control-plane host (not `127.0.0.1`)
        
    - Bound to an **actual interface** on your machine (e.g. `eth0`, `ens3`, etc.)
        

> In your case:

bash

CopyEdit

`10.8.8.10`

This is likely the **host IP on your local or virtual network**.

---

## 📁 What Does This Command Generate?

|File / Dir|Purpose|
|---|---|
|`/etc/kubernetes/admin.conf`|Kubeconfig used by admin (`kubectl`)|
|`/etc/kubernetes/manifests/`|Static pod manifests for control plane components|
|`/var/lib/kubelet/`|Kubelet configuration + certificates|
|`/etc/kubernetes/pki/`|TLS certs used for secure cluster comms|

---

## 🚀 What Happens Internally?

1. Pre-flight checks (ports, swap, CRI, etc.)
    
2. Pulls required images (`pause`, `etcd`, `apiserver`, etc.)
    
3. Generates TLS certs and kubeconfigs
    
4. Starts static pods via the kubelet
    
5. Waits for control plane health (`curl 127.0.0.1:10248/healthz`)
    
6. Sets up `kubeadm join` token
    
7. Outputs post-installation instructions
    

---

## 🔑 When Would You Modify These Flags?

|Scenario|Change you’d make|
|---|---|
|Using Flannel|`--pod-network-cidr=10.244.0.0/16`|
|Running in a different subnet|Change `--apiserver-advertise-address`|
|Single-node cluster with Cilium|Keep current values ✅|
|Multi-node production setup|You might add `--control-plane-endpoint`|
|Using HA setup|Use load balancer IP as `--advertise-address`|

---

## 🧪 Bonus: View These Settings Later

bash

CopyEdit

`cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep advertise-address`