# Certified Kubernetes Administrator (CKA) – Comprehensive Command & Concept Sheet

_(Aligned to CNCF CKA v1.32 curriculum)_

---

## 0 30‑Second Exam Warm‑Up

```bash
alias k=kubectl                      # if not pre‑set
k config use-context <ctx>           # switch to specified context
k config set-context --current --namespace=<ns>  # set namespace
export ETCDCTL_API=3                 # for etcdctl commands
export do="--dry-run=client -o yaml" # for quick manifests
```

Docs tab → `kubernetes.io/docs` + ctrl‑F search for quick access.

---

## 1 Cluster Architecture, Installation & Configuration (25%)

### 1.1 Bootstrap with kubeadm

```bash
# Single‑CP or HA control‑plane
sudo kubeadm init \
  --control-plane-endpoint "k8s-lb:6443" \       # VIP / LB DNS
  --upload-certs \                                # share cert key for extra CPs
  --pod-network-cidr 10.244.0.0/16 \
  --apiserver-advertise-address $(hostname -i)

# Join worker node
sudo kubeadm join k8s-lb:6443 --token <tkn> \
  --discovery-token-ca-cert-hash sha256:<hash>
  
# Join extra control plane
sudo kubeadm join k8s-lb:6443 --token <tkn> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>

# Generate new join token
kubeadm token create --print-join-command
```

### 1.2 Lifecycle & etcd

```bash
# Upgrade path
sudo kubeadm upgrade plan
sudo apt-get update
sudo apt-get install -y kubeadm=1.29.2-00
sudo kubeadm upgrade apply v1.29.2

# For worker nodes
sudo apt-get install -y kubelet=1.29.2-00 kubectl=1.29.2-00
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd.bak \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd.bak \
  --data-dir /var/lib/etcd-from-backup
  
# Update etcd pod manifest after restore
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Change --data-dir to /var/lib/etcd-from-backup
```

### 1.3 Extension Interfaces

|Interface|What it does|Common implementation|
|---|---|---|
|**CNI**|Container networking|`kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml`|
|**CSI**|Storage management|vendor‑specific YAML (e.g., AWS EBS CSI)|
|**CRI**|Container runtime|containerd: `--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock`|
|**OCI**|Container image format|Docker, containerd, CRI-O|

### 1.4 RBAC Configuration

```bash
# Create role
kubectl create role pod-reader --verb=get,list,watch --resource=pods,pods/log

# Create role binding
kubectl create rolebinding read-pods --role=pod-reader --user=jane

# Create cluster role
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes

# Create cluster role binding
kubectl create clusterrolebinding read-nodes --clusterrole=node-reader --user=admin --group=admins

# Service accounts
kubectl create serviceaccount backend-sa
kubectl create rolebinding backend-rb --role=backend-role --serviceaccount=default:backend-sa

# Verify permissions
kubectl auth can-i list pods --as jane
kubectl auth can-i create deployments --as system:serviceaccount:default:backend-sa
```

### 1.5 HA Control Plane Configuration

```bash
# External etcd topology (recommended for production)
sudo kubeadm init --control-plane-endpoint "lb.example.com:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --external-etcd-endpoints=https://etcd-0:2379,https://etcd-1:2379,https://etcd-2:2379 \
  --etcd-cafile=/etc/etcd/ca.crt \
  --etcd-certfile=/etc/etcd/apiserver-etcd-client.crt \
  --etcd-keyfile=/etc/etcd/apiserver-etcd-client.key

# Stacked etcd topology (simpler setup)
sudo kubeadm init --control-plane-endpoint "lb.example.com:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Check control plane health
kubectl get nodes -l node-role.kubernetes.io/control-plane=''
kubectl -n kube-system get pods -l tier=control-plane
```

---

## 2 Workloads & Scheduling (15%)

### 2.1 Core Deployment Commands

```bash
# Create & manage deployments
kubectl create deploy nginx --image=nginx:1.21 --replicas=3

# Scale resources
kubectl scale deploy nginx --replicas=5
kubectl autoscale deploy nginx --min=3 --max=10 --cpu-percent=80

# Update image
kubectl set image deploy/nginx nginx=nginx:1.22 --record
kubectl rollout status deploy/nginx
kubectl rollout history deploy/nginx
kubectl rollout undo deploy/nginx [--to-revision=2]

# Pause/resume rollout
kubectl rollout pause deploy/nginx
kubectl rollout resume deploy/nginx

# Access containers
kubectl exec -it pod-name -- sh
kubectl logs -f pod-name -c container-name
```

### 2.2 Pod Scheduling Controls

```bash
# Node selector (basic)
spec:
  nodeSelector:
    disktype: ssd

# Taints and tolerations
## Add taint to node
kubectl taint nodes worker1 key=value:NoSchedule

## Pod toleration
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"

# Node affinity (hard requirement)
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: [us-east-1a, us-east-1b]

# Node affinity (soft preference)
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values: [compute]

# Pod affinity (co-locate with app=web pods)
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["web"]
        topologyKey: kubernetes.io/hostname
```

### 2.3 Self‑Healing & Priority

```yaml
# Readiness probe (HTTP)
spec:
  containers:
  - name: app
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10

# Liveness probe (TCP)
spec:
  containers:
  - name: app
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
      
# Startup probe (exec command)
spec:
  containers:
  - name: app
    startupProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 30
```

```bash
# Create PriorityClass
kubectl create priorityclass high --value=1000000 --global-default=false --description="Critical pods"

# Use in pod spec
spec:
  priorityClassName: high
```

### 2.4 ConfigMaps and Secrets

```bash
# Create ConfigMap
kubectl create configmap app-config --from-literal=DB_HOST=mysql --from-file=settings.conf

# Create Secret
kubectl create secret generic db-creds --from-literal=username=admin --from-literal=password=secret

# Use as environment variables
spec:
  containers:
  - name: app
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    - name: DATABASE_USER
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: username

# Use as volume
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

### 2.5 Horizontal Pod Autoscaler (HPA)

```bash
# Create HPA
kubectl autoscale deploy nginx --min=3 --max=20 --cpu-percent=65

# YAML example for CPU + memory
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

# Check status
kubectl get hpa
kubectl describe hpa web-app
```

**Metrics-Server** (if showing `unkonwn`):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system \
  --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

---

## 3 Services & Networking (20%)

### 3.1 Service Types

```bash
# ClusterIP (default) - internal cluster access only
kubectl expose deploy web --port=80 --target-port=8080

# NodePort - expose on each node's IP at static port
kubectl expose deploy web --port=80 --target-port=8080 --type=NodePort --node-port=30080

# LoadBalancer - external load balancer (cloud only)
kubectl expose deploy web --port=80 --target-port=8080 --type=LoadBalancer

# ExternalName - CNAME DNS record to external service
apiVersion: v1
kind: Service
metadata:
  name: external-svc
spec:
  type: ExternalName
  externalName: my.database.example.com
```

### 3.2 Network Policies

```yaml
# Allow traffic from frontend pods to backend pods on port 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
      
# Deny all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  
# Allow all egress traffic to specific CIDR
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-cidr
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
```

### 3.3 Ingress & Gateway API

```yaml
# NGINX ingress controller install
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Basic Ingress resource
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
```

```yaml
# Gateway API (newer API for traffic management)
# Install CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# Create gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gw
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
      
# HTTPRoute for traffic routing
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: prod-gw
  hostnames:
  - "webapp.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-svc
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-svc
      port: 80
```

### 3.4 DNS & Service Discovery

```bash
# Verify CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns

# DNS resolution within pods
kubectl exec -it busybox -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it busybox -- nslookup web-svc.default.svc.cluster.local

# Basic DNS patterns:
# <service-name>.<namespace>.svc.cluster.local
# <pod-ip-with-dashes>.<namespace>.pod.cluster.local
```

### 3.5 Network Troubleshooting

```bash
# Check service endpoints
kubectl get endpoints web-svc
kubectl describe svc web-svc

# Debug connectivity
kubectl run --rm -it net-debug --image=nicolaka/netshoot -- /bin/bash
  # Tools available: ping, curl, dig, nslookup, ip, tcpdump, etc.

# Check if service is accessible
kubectl exec -it busybox -- curl -m2 http://web-svc:80/

# View pod networking details
kubectl get pods web-pod -o yaml | grep -A 10 podIP

# Test NetworkPolicy
kubectl run --rm -it policy-test --image=busybox --labels=role=frontend \
  -- wget -T 2 -O- http://backend-svc:8080/
```

---

## 4 Storage (10%)

### 4.1 Volume Types

```yaml
# EmptyDir - ephemeral storage
spec:
  containers:
  - name: app
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  volumes:
  - name: cache-volume
    emptyDir: {}

# HostPath - mounts from host filesystem (not recommended for production)
spec:
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory

# ConfigMap volume
spec:
  volumes:
  - name: config-volume
    configMap:
      name: app-config

# Secret volume
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secrets
```

### 4.2 PVC Access Modes

|Mode|Description|Use case|
|---|---|---|
|`ReadWriteOnce` (RWO)|Read-write by single node|Most cloud volumes|
|`ReadWriteOncePod` (RWOP)|Read-write by single pod|Stricter than RWO|
|`ReadWriteMany` (RWX)|Read-write by multiple nodes|NFS, CephFS|
|`ReadOnlyMany` (ROX)|Read-only by multiple nodes|Config data|

### 4.3 Storage Classes

```yaml
# Local storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete

# Cloud provisioner example (AWS)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  fsType: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### 4.4 PVC & PV

```yaml
# Create PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: fast
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

# Static PV (for pre-provisioned volume)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /exports
    server: nfs-server.example.com

# Usage in pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /var/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
```

### 4.5 Volume Operations

```bash
# View PV/PVC status
kubectl get pv,pvc
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name>

# Expand volume (if supported)
kubectl edit pvc app-data  # Change spec.resources.requests.storage

# Reclaim policy options (for PV)
# - Delete: volume deleted when PVC deleted
# - Retain: volume kept but must be manually reclaimed
# - Recycle: data deleted but volume kept (deprecated)
```

---

## 5 Helm & Kustomize

### 5.1 Helm Basics

```bash
# Add repo and install chart
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install redis bitnami/redis -n app --create-namespace

# List releases
helm list -A

# Upgrade release with values
helm upgrade redis bitnami/redis -n app --set auth.password=new-password
helm upgrade --reuse-values redis bitnami/redis -n app --set persistence.size=10Gi

# Rollback to previous release
helm rollback redis 1 -n app

# Uninstall release
helm uninstall redis -n app

# Common applications via Helm
# Prometheus Monitoring
helm install prom prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# Cert-Manager for TLS certificates
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

### 5.2 Kustomize

```bash
# Sample directory structure
# base/
#   - deployment.yaml
#   - service.yaml
#   - kustomization.yaml
# overlays/
#   - prod/
#     - kustomization.yaml
#     - patch-replicas.yaml
#   - dev/
#     - kustomization.yaml
#     - patch-resources.yaml

# Base kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml

# Overlay kustomization.yaml (prod)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
nameSuffix: -prod
namespace: production
patchesStrategicMerge:
  - patch-replicas.yaml
images:
  - name: myapp
    newName: registry.example.com/myapp
    newTag: v1.2.3

# Apply kustomization
kubectl apply -k overlays/prod/
kubectl kustomize overlays/prod/  # Preview without applying
```

---

## 6 CRDs & Operators

```yaml
# Example Custom Resource Definition
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

```bash
# List CRDs in cluster
kubectl get crd

# Install an operator (example: cert-manager)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml

# Verify operator pods
kubectl get pods -n cert-manager

# Use custom resource from operator
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: user@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

---

## 7 Troubleshooting (30%)

### 7.1 Node Troubleshooting

```bash
# Node status and details
kubectl get nodes
kubectl describe node <node-name>
kubectl get node <node-name> -o yaml

# Check node resource usage
kubectl top node
kubectl describe node <node-name> | grep -A 10 Allocated

# Check kubelet status
ssh <node>
systemctl status kubelet
journalctl -u kubelet -f
ls -la /var/log/pods/

# Common kubelet issues
## Check certificate issues
openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -text

## Check kubelet config
cat /var/lib/kubelet/config.yaml
cat /etc/kubernetes/kubelet.conf

## Check node registration issues
cat /etc/hosts  # Ensure hostname resolution works

# Memory/disk pressure issues
df -h  # Check disk space
free -m  # Check memory
```

### 7.2 Control Plane Troubleshooting

```bash
# Check control plane component health
kubectl get pods -n kube-system
kubectl -n kube-system logs kube-apiserver-master -f
kubectl -n kube-system logs kube-controller-manager-master -f
kubectl -n kube-system logs kube-scheduler-master -f

# If components run as static pods
ls -la /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Check certificates
cd /etc/kubernetes/pki
openssl x509 -in apiserver.crt -text -noout | grep "Not After"

# Check etcd health
kubectl -n kube-system exec -it etcd-master -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Component logs on the node
journalctl -u kubelet -f
docker logs kube-apiserver-master  # if using Docker
crictl logs <container-id>  # if using containerd
```

### 7.3 Pod & Application Troubleshooting

```bash
# Check pod status
kubectl get pods
kubectl describe pod <pod-name>

# Check container logs
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> -c <container-name> --previous  # for crashed containers

# Debug failing pods
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get pod <pod-name> -o yaml > pod-def.yaml  # examine YAML

# Common pod issues
## ImagePullBackOff/ErrImagePull
kubectl describe pod <pod-name> | grep -A10 Events  # Check image name, registry access
echo $KUBECONFIG  # Check your credentials if using private registry

## CrashLoopBackOff
kubectl logs <pod-name> --previous  # Check why container is crashing
kubectl exec -it <pod-name> -- /bin/sh  # Try interactive debugging

## Check resource constraints
kubectl describe pod <pod-name> | grep -A5 Limits
kubectl top pods

# Create debug pod in same namespace as failing pod
kubectl run debug --rm -it --image=busybox -- /bin/sh
```

### 7.4 Service & Networking Troubleshooting

```bash
# Check service resolution
kubectl get svc <service-name>
kubectl describe svc <service-name>

# Check endpoints
kubectl get endpoints <service-name>
kubectl get pods -o wide | grep <selector-label>  # Find pods targeted by service

# DNS troubleshooting
kubectl exec -it <pod-name> -- nslookup kubernetes.default
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns

# NetworkPolicy issues
kubectl get networkpolicy
kubectl describe networkpolicy <policy-name>

# Test reachability
kubectl run test-$RANDOM --rm -it --image=nicolaka/netshoot -- bash
  # Then run: curl <service-name>.<namespace>.svc.cluster.local

# Create network debug pod
kubectl run tmp-shell --rm -it --image=nicolaka/netshoot -- bash
# Tools: dig, nslookup, curl, tcpdump, ping, traceroute, etc.
```

### 7.5 Resource Monitoring

```bash
# Node resources
kubectl top node
kubectl describe node <node-name> | grep -A 10 "Allocated resources"

# Pod resources 
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Container resource utilization
kubectl top pod <pod-name> --containers

# Install metrics-server if not present
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## 8 Quick Resource Short‑Names

```
po pods  svc services  ing ingress  netpol networkpolicies  hpa horizontalpodautoscalers
rs replicaset  deploy deployment  ds daemonset  sts statefulset  cm configmap  secret
ns namespace  pvc persistentvolumeclaim  pv persistentvolume  sc storageclasses
sa serviceaccount  no node  crd customresourcedefinition  rb rolebinding  role role
```

---

## 9 Exam Tips & Tricks

- Set up persistent aliases at start: `alias k=kubectl` and `export KUBECONFIG=/root/.kube/config`
- Check which namespace resources are in: `k get all -A | grep <resource-name>`
- Use `--dry-run=client -o yaml > file.yaml` to quickly create resource definitions
- Use `k explain pod.spec.containers --recursive` for quick documentation
- For multi-part questions, don't get stuck - move on and return later
- Time management: aim for 10-12 questions per hour (about 5 minutes per question)
- Use `k get events --sort-by=.metadata.creationTimestamp` to debug recent issues
- Remember imperative commands to save time over creating YAML manually

---

### End of Sheet – Good luck with your CKA exam!