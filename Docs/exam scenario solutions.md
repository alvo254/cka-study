

# Comprehensive CKA Exam Scenario Solutions

## Table of Contents

- [Storage (10%)]
    - [Question 1: Dynamic Volume Provisioning]
    - [Question 2: Volume Management]
- [Workloads and Scheduling (15%)]
    - [Question 3: Deployment Rollout and Rollback]
    - [Question 4: ConfigMaps and Secrets]
    - [Question 5: Pod Scheduling and Affinity]
- [Cluster Architecture, Installation and Configuration (25%)]
    - [Question 6: Kubeadm Cluster Setup]
    - [Question 7: RBAC Configuration]
    - [Question 8: Cluster Upgrades]
    - [Question 9: Custom Resource Definitions]
- [Services and Networking (20%)]
    - [Question 10: Network Policies]
    - [Question 11: Service Configuration]
    - [Question 12: Gateway API Configuration]
    - [Question 13: DNS Configuration]
- [Troubleshooting (30%)]
    - [Question 14: Node Troubleshooting]
    - [Question 15: Cluster Component Troubleshooting]
    - [Question 16: Application Resource Monitoring]
    - [Question 17: Container Output Troubleshooting]
    - [Question 18: Network Troubleshooting]
    - [Question 19: Multi-Component Troubleshooting]
    - [Question 20: Resource Management and Quota Troubleshooting]

## Storage (10%)

### Question 1: Dynamic Volume Provisioning

**Task:**

1. Create a StorageClass named `fast-storage` that uses the `standard` provisioner
2. Configure the StorageClass with `reclaimPolicy: Retain`
3. Create a PersistentVolumeClaim named `db-data` in the `database` namespace that requests 10Gi of storage using this StorageClass
4. Create a Pod named `db-pod` that uses this PersistentVolumeClaim mounted at `/var/lib/mysql`

**Solution:**

First, create the namespace if it doesn't exist:

```bash
kubectl create namespace database
```

Create a StorageClass with the specified requirements:

```yaml
# fast-storage-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/standard  # This is the built-in provisioner
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

Apply the StorageClass:

```bash
kubectl apply -f fast-storage-sc.yaml
```

Next, create the PersistentVolumeClaim in the database namespace:

```yaml
# db-data-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
  namespace: database
spec:
  storageClassName: fast-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Apply the PVC:

```bash
kubectl apply -f db-data-pvc.yaml
```

Finally, create a Pod that uses this PVC:

```yaml
# db-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  namespace: database
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    volumeMounts:
    - name: mysql-data
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-data
    persistentVolumeClaim:
      claimName: db-data
```

Apply the Pod:

```bash
kubectl apply -f db-pod.yaml
```

To verify everything is working correctly:

```bash
kubectl get storageclass fast-storage
kubectl get pvc db-data -n database
kubectl get pod db-pod -n database
```

### Question 2: Volume Management

**Task:**

1. Identify the current status of the PersistentVolume `legacy-data`
2. If it's bound to a claim, delete the claim (but not any associated pods)
3. Modify the PersistentVolume to change its reclaim policy to `Recycle`
4. Create a new PersistentVolumeClaim named `app-data` in the `default` namespace that can bind to this volume

**Solution:**

First, check the status of the PersistentVolume:

```bash
kubectl get pv legacy-data
```

If it's bound to a claim, find which claim:

```bash
kubectl get pv legacy-data -o jsonpath='{.spec.claimRef.name}'
```

Let's assume it's bound to a claim named `old-claim` in the `some-namespace` namespace. Delete that claim:

```bash
kubectl delete pvc old-claim -n some-namespace
```

Now, patch the PV to change its reclaim policy to `Recycle`:

```bash
kubectl patch pv legacy-data -p '{"spec":{"persistentVolumeReclaimPolicy":"Recycle"}}'
```

Alternatively, you can edit the PV directly:

```bash
kubectl edit pv legacy-data
```

Then change the `persistentVolumeReclaimPolicy` field to `Recycle`.

Now, create a new PVC that can bind to this volume:

```yaml
# app-data-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce  # This should match the access mode of legacy-data
  resources:
    requests:
      storage: 5Gi  # This should be less than or equal to the capacity of legacy-data
  # If legacy-data has a storageClassName, include it here
  # storageClassName: some-storage-class
  # If legacy-data has specific selectors, include them here to ensure binding
  selector:
    matchLabels:
      pv-name: legacy-data
```

Apply the PVC:

```bash
kubectl apply -f app-data-pvc.yaml
```

To verify it's correctly bound:

```bash
kubectl get pvc app-data
kubectl get pv legacy-data
```

## Workloads and Scheduling (15%)

### Question 3: Deployment Rollout and Rollback

**Task:**

1. Check the rollout history of the deployment
2. Roll back the deployment to the previous stable version
3. Confirm the application is running with 3 replicas
4. Configure the deployment to use the `RollingUpdate` strategy with `maxSurge: 1` and `maxUnavailable: 0`

**Solution:**

First, check the rollout history:

```bash
kubectl rollout history deployment web-app -n production
```

To see the details of a specific revision:

```bash
kubectl rollout history deployment web-app -n production --revision=2
```

Roll back to the previous version:

```bash
kubectl rollout undo deployment web-app -n production
```

If you want to roll back to a specific revision:

```bash
kubectl rollout undo deployment web-app -n production --to-revision=2
```

Confirm the application is running with 3 replicas:

```bash
kubectl get deployment web-app -n production
```

Now, configure the deployment to use the RollingUpdate strategy with the specified parameters:

```bash
kubectl patch deployment web-app -n production -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
```

Alternatively, you can use `kubectl edit`:

```bash
kubectl edit deployment web-app -n production
```

Then modify the strategy section to:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Verify the changes:

```bash
kubectl describe deployment web-app -n production
```

### Question 4: ConfigMaps and Secrets

**Task:**

1. Create a ConfigMap named `app-config` with the following data:
    - `DB_HOST=db.example.com`
    - `DB_PORT=5432`
    - `APP_MODE=production`
2. Create a Secret named `app-secrets` with the following data:
    - `DB_USER=admin`
    - `DB_PASSWORD=s3cret`
3. Create a Deployment named `backend-app` that mounts both the ConfigMap and Secret as environment variables

**Solution:**

Create the ConfigMap:

```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=db.example.com \
  --from-literal=DB_PORT=5432 \
  --from-literal=APP_MODE=production
```

Create the Secret:

```bash
kubectl create secret generic app-secrets \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cret
```

Create the Deployment:

```yaml
# backend-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-app
  template:
    metadata:
      labels:
        app: backend-app
    spec:
      containers:
      - name: backend
        image: nginx:latest  # Change to your actual backend app image
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
```

Apply the Deployment:

```bash
kubectl apply -f backend-app-deployment.yaml
```

Verify the configuration:

```bash
kubectl get configmap app-config
kubectl get secret app-secrets
kubectl get deployment backend-app
kubectl describe deployment backend-app
```

To check that environment variables are properly set, you can exec into one of the pods:

```bash
kubectl exec -it $(kubectl get pods -l app=backend-app -o jsonpath="{.items[0].metadata.name}") -- env | grep -E 'DB_|APP_'
```

### Question 5: Pod Scheduling and Affinity

**Task:**

1. Label one of your worker nodes with `disk=ssd`
2. Create a Pod named `data-processor` that will be scheduled only on nodes with the `disk=ssd` label
3. Create another Pod named `web-frontend` that should preferably run on nodes with the `disk=ssd` label, but can run elsewhere if necessary
4. Create a Pod named `monitoring-agent` that must not be co-located with either of the above pods

**Solution:**

First, label one of the worker nodes:

```bash
# List all worker nodes
kubectl get nodes

# Label a specific node
kubectl label node <worker-node-name> disk=ssd
```

Create the `data-processor` Pod with a node selector:

```yaml
# data-processor-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  containers:
  - name: data-processor
    image: nginx:latest  # Replace with your actual image
  nodeSelector:
    disk: ssd
```

Apply it:

```bash
kubectl apply -f data-processor-pod.yaml
```

Create the `web-frontend` Pod with node affinity:

```yaml
# web-frontend-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-frontend
  labels:
    app: web-frontend
spec:
  containers:
  - name: web-frontend
    image: nginx:latest  # Replace with your actual image
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
```

Apply it:

```bash
kubectl apply -f web-frontend-pod.yaml
```

Create the `monitoring-agent` Pod with pod anti-affinity:

```yaml
# monitoring-agent-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: monitoring-agent
  labels:
    app: monitoring-agent
spec:
  containers:
  - name: monitoring-agent
    image: nginx:latest  # Replace with your actual image
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web-frontend
        topologyKey: "kubernetes.io/hostname"
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - data-processor
        topologyKey: "kubernetes.io/hostname"
```

Apply it:

```bash
kubectl apply -f monitoring-agent-pod.yaml
```

Verify that the pods are scheduled correctly:

```bash
kubectl get pods -o wide
```

## Cluster Architecture, Installation and Configuration (25%)

### Question 6: Kubeadm Cluster Setup

**Task:**

1. Create a kubeadm configuration file that:
    - Uses the `172.16.0.0/16` pod CIDR range
    - Sets the control plane endpoint to `k8s-master.example.com:6443`
    - Uses a specific version of Kubernetes (e.g., `v1.26.0`)
2. Initialize the control plane using this configuration
3. Configure kubectl for the admin user
4. Install a CNI plugin of your choice
5. Prepare a join command for worker nodes

**Solution:**

First, create a kubeadm configuration file:

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: <control-plane-ip>
  bindPort: 6443
nodeRegistration:
  name: <control-plane-hostname>
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.26.0
controlPlaneEndpoint: "k8s-master.example.com:6443"
networking:
  podSubnet: "172.16.0.0/16"
  serviceSubnet: "10.96.0.0/12"
```

Initialize the control plane:

```bash
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs
```

Configure kubectl for the admin user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install a CNI plugin (Calico in this example):

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

The kubeadm init command output will have the join command for worker nodes. It looks like:

```bash
kubeadm join k8s-master.example.com:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

If you need to generate a new token:

```bash
kubeadm token create --print-join-command
```

Verify the cluster is up and running:

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

### Question 7: RBAC Configuration

**Task:**

1. Create a new ServiceAccount named `developer-account` in the `development` namespace
2. Create a Role that allows the following permissions in the `development` namespace only:
    - Get, list, and watch pods
    - Create and delete deployments
    - Update and patch services
3. Bind this Role to the ServiceAccount
4. Create a ClusterRole that allows viewing resources across all namespaces
5. Bind this ClusterRole to the same ServiceAccount

**Solution:**

First, create the namespace if it doesn't exist:

```bash
kubectl create namespace development
```

Create the ServiceAccount:

```bash
kubectl create serviceaccount developer-account -n development
```

Create the Role:

```yaml
# developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: development
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "delete"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["update", "patch"]
```

Apply the Role:

```bash
kubectl apply -f developer-role.yaml
```

Create the RoleBinding:

```yaml
# developer-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: development
subjects:
- kind: ServiceAccount
  name: developer-account
  namespace: development
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

Apply the RoleBinding:

```bash
kubectl apply -f developer-rolebinding.yaml
```

Create the ClusterRole:

```yaml
# viewer-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: viewer-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods", "services", "namespaces", "configmaps", "endpoints", "persistentvolumeclaims", "events", "nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets", "replicasets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]
```

Apply the ClusterRole:

```bash
kubectl apply -f viewer-clusterrole.yaml
```

Create the ClusterRoleBinding:

```yaml
# viewer-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: viewer-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: developer-account
  namespace: development
roleRef:
  kind: ClusterRole
  name: viewer-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

Apply the ClusterRoleBinding:

```bash
kubectl apply -f viewer-clusterrolebinding.yaml
```

Verify the RBAC configuration:

```bash
kubectl get serviceaccount developer-account -n development
kubectl get role developer-role -n development
kubectl get rolebinding developer-rolebinding -n development
kubectl get clusterrole viewer-clusterrole
kubectl get clusterrolebinding viewer-clusterrolebinding
```

Test the permissions:

```bash
kubectl auth can-i get pods --as=system:serviceaccount:development:developer-account -n development
kubectl auth can-i create deployments --as=system:serviceaccount:development:developer-account -n development
kubectl auth can-i update services --as=system:serviceaccount:development:developer-account -n development
kubectl auth can-i get pods --as=system:serviceaccount:development:developer-account -n default
```

### Question 8: Cluster Upgrades

**Task:**

1. Check the current version of all components
2. Upgrade the kubeadm tool to the target version
3. Create a plan for upgrading the control plane node
4. Execute the upgrade of the control plane components
5. Update the kubelet and kubectl on the control plane node
6. Drain a worker node in preparation for its upgrade

**Solution:**

First, check the current version of all components:

```bash
kubectl version --short
kubectl get nodes
kubeadm version
kubelet --version
```

Upgrade the kubeadm tool to the target version (let's say v1.27.0):

```bash
# Update package repository
sudo apt update

# Install the specific version of kubeadm
sudo apt-get install -y --allow-change-held-packages kubeadm=1.27.0-00
```

Create a plan for upgrading the control plane node:

```bash
sudo kubeadm upgrade plan
```

Execute the upgrade of the control plane components:

```bash
sudo kubeadm upgrade apply v1.27.0
```

Update the kubelet and kubectl on the control plane node:

```bash
# Install the specific version of kubelet and kubectl
sudo apt-get install -y --allow-change-held-packages kubelet=1.27.0-00 kubectl=1.27.0-00

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Drain a worker node in preparation for its upgrade:

```bash
# Get the list of nodes
kubectl get nodes

# Drain the worker node
kubectl drain <worker-node-name> --ignore-daemonsets --delete-emptydir-data
```

Once the worker node is drained, you can SSH to it and update kubeadm, kubelet, and kubectl:

```bash
# On the worker node
sudo apt-get install -y --allow-change-held-packages kubeadm=1.27.0-00
sudo kubeadm upgrade node
sudo apt-get install -y --allow-change-held-packages kubelet=1.27.0-00 kubectl=1.27.0-00
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Finally, uncordon the worker node:

```bash
kubectl uncordon <worker-node-name>
```

Verify the upgrade:

```bash
kubectl get nodes
```

### Question 9: Custom Resource Definitions

**Task:**

1. Create a Custom Resource Definition (CRD) named `backups.example.com` with:
    - Group: `example.com`
    - Version: `v1`
    - Scope: `Namespaced`
    - Validation schema requiring `schedule` and `destination` fields
2. Create a custom resource object of this type
3. Install a simple operator (such as the CertManager operator) using Helm

**Solution:**

Create the CRD:

```yaml
# backups-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.example.com
spec:
  group: example.com
  names:
    plural: backups
    singular: backup
    kind: Backup
    shortNames:
    - bkp
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        required: ["spec"]
        properties:
          spec:
            type: object
            required: ["schedule", "destination"]
            properties:
              schedule:
                type: string
                description: "Backup schedule in cron format"
              destination:
                type: string
                description: "Backup destination path or URL"
              retention:
                type: integer
                description: "Number of days to retain the backup"
    additionalPrinterColumns:
    - name: Schedule
      type: string
      jsonPath: .spec.schedule
    - name: Destination
      type: string
      jsonPath: .spec.destination
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

Apply the CRD:

```bash
kubectl apply -f backups-crd.yaml
```

Create a custom resource object:

```yaml
# database-backup.yaml
apiVersion: example.com/v1
kind: Backup
metadata:
  name: database-backup
spec:
  schedule: "0 1 * * *"
  destination: "s3://my-backup-bucket/db"
  retention: 7
```

Apply the custom resource:

```bash
kubectl apply -f database-backup.yaml
```

Install the cert-manager operator using Helm:

```bash
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update Helm repositories
helm repo update

# Install cert-manager with CRDs
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0 \
  --set installCRDs=true
```

Verify everything is working:

```bash
kubectl get crd | grep example.com
kubectl get backups
kubectl get pods -n cert-manager
```

## Services and Networking (20%)

### Question 10: Network Policies

**Task:**

1. Create a namespace called `secure-apps`
2. Deploy a pod named `backend-api` with label `app=backend` in this namespace
3. Deploy a pod named `frontend-ui` with label `app=frontend` in this namespace
4. Create a NetworkPolicy named `api-policy` that:
    - Allows ingress traffic to `backend-api` only from `frontend-ui`
    - Allows egress traffic from `backend-api` only to pods with label `db=true`
    - Blocks all other traffic to and from the `backend-api`

**Solution:**

Create the namespace:

```bash
kubectl create namespace secure-apps
```

Deploy the backend-api pod:

```yaml
# backend-api-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-api
  namespace: secure-apps
  labels:
    app: backend
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Apply it:

```bash
kubectl apply -f backend-api-pod.yaml
```

Deploy the frontend-ui pod:

```yaml
# frontend-ui-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-ui
  namespace: secure-apps
  labels:
    app: frontend
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Apply it:

```bash
kubectl apply -f frontend-ui-pod.yaml
```

Create the NetworkPolicy:

```yaml
# api-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: secure-apps
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          db: "true"
```

Apply the NetworkPolicy:

```bash
kubectl apply -f api-policy.yaml
```

Verify the NetworkPolicy:

```bash
kubectl get networkpolicy -n secure-apps
kubectl describe networkpolicy api-policy -n secure-apps
```

Test the NetworkPolicy by deploying a test pod and attempting to access the backend:

```bash
# Create a test pod
kubectl run test-pod -n secure-apps --image=alpine --restart=Never -- sleep 3600

# Attempt to connect to the backend-api (should fail)
kubectl exec -it test-pod -n secure-apps -- wget --timeout=5 backend-api

# Create a test DB pod
kubectl run db-pod -n secure-apps --image=mysql:5.7 --labels="db=true" --env="MYSQL_ROOT_PASSWORD=password" --restart=Never
```

### Question 11: Service Configuration

**Task:**

1. Create a Deployment named `web-server` with 3 replicas running nginx
2. Expose this deployment within the cluster using a ClusterIP service named `web-internal`
3. Create a NodePort service named `web-external` that exposes the deployment on port 30080
4. Create a LoadBalancer service named `web-lb` that exposes the deployment to external traffic
5. Verify connectivity to all three services

**Solution:**

Create the deployment:

```yaml
# web-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Apply the deployment:

```bash
kubectl apply -f web-server-deployment.yaml
```

Create the ClusterIP service:

```yaml
# web-internal-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-internal
spec:
  selector:
    app: web-server
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

Apply the ClusterIP service:

```bash
kubectl apply -f web-internal-service.yaml
```

Create the NodePort service:

```yaml
# web-external-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-external
spec:
  selector:
    app: web-server
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

Apply the NodePort service:

```bash
kubectl apply -f web-external-service.yaml
```

Create the LoadBalancer service:

```yaml
# web-lb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  selector:
    app: web-server
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

Apply the LoadBalancer service:

```bash
kubectl apply -f web-lb-service.yaml
```

Verify connectivity to all three services:

```bash
# Get services
kubectl get services

# Test ClusterIP service (from within the cluster)
kubectl run test-pod --image=nginx:alpine --restart=Never
kubectl exec -it test-pod -- curl web-internal

# Test NodePort service (from node IP)
curl <Node-IP>:30080

# Test LoadBalancer service (if cloud provider supports it)
# Get the external IP
kubectl get svc web-lb
curl <External-IP>
```

## Question 12: Gateway API Configuration

**Task:**

1. Install the Gateway API CRDs in your cluster
2. Create a GatewayClass named `production-class`
3. Create a Gateway named `main-gateway` using this GatewayClass that listens on port 80
4. Create an HTTPRoute named `web-route` that routes traffic with the path `/app` to the `web-server` service
5. Verify the Gateway and HTTPRoute are properly configured

**Solution:**

First, let's install the Gateway API CRDs in the cluster:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.7.0/standard-install.yaml
```

Now, let's create the GatewayClass. This is a cluster-wide resource that defines a type of Gateway with a specific controller implementation:

```yaml
# production-class.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: production-class
spec:
  controllerName: example.com/gateway-controller
```

Apply the GatewayClass:

```bash
kubectl apply -f production-class.yaml
```

Next, let's create the Gateway that listens on port 80:

```yaml
# main-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: production-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

Apply the Gateway:

```bash
kubectl apply -f main-gateway.yaml
```

Now, let's create the HTTPRoute that routes traffic with the path `/app` to the `web-server` service:

```yaml
# web-route.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: main-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app
    backendRefs:
    - name: web-server
      port: 80
```

Apply the HTTPRoute:

```bash
kubectl apply -f web-route.yaml
```

Finally, let's verify that everything is configured correctly:

```bash
# Check the GatewayClass
kubectl get gatewayclass production-class

# Check the Gateway
kubectl get gateway main-gateway
kubectl describe gateway main-gateway

# Check the HTTPRoute
kubectl get httproute web-route
kubectl describe httproute web-route
```

If everything is working correctly, you should see the Gateway and HTTPRoute in a healthy state, and traffic to the path `/app` should be routed to the `web-server` service.

## Question 13: DNS Configuration

**Task:**

1. Inspect the CoreDNS configuration in your cluster
2. Create a service named `custom-app` in the `apps` namespace
3. Create a Pod that can resolve this service using DNS
4. Modify the CoreDNS configuration to implement a custom domain rewrite rule
5. Verify the DNS changes are working correctly

**Solution:**

First, let's inspect the CoreDNS configuration in the cluster:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

This will show you the current CoreDNS configuration, including the Corefile, which defines how DNS queries are processed.

Now, let's create the `apps` namespace if it doesn't exist:

```bash
kubectl create namespace apps
```

Let's create a service named `custom-app` in the `apps` namespace. We'll also create a deployment to back this service:

```yaml
# custom-app.yaml
apiVersion: v1
kind: Service
metadata:
  name: custom-app
  namespace: apps
spec:
  selector:
    app: custom-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-app
  namespace: apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-app
  template:
    metadata:
      labels:
        app: custom-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Apply the service and deployment:

```bash
kubectl apply -f custom-app.yaml
```

Next, let's create a Pod that can resolve this service using DNS:

```yaml
# dns-client.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-client
spec:
  containers:
  - name: dns-client
    image: busybox:latest
    command:
    - sleep
    - "3600"
```

Apply the Pod:

```bash
kubectl apply -f dns-client.yaml
```

Once the Pod is running, we can test DNS resolution:

```bash
kubectl exec -it dns-client -- nslookup custom-app.apps.svc.cluster.local
```

Now, let's modify the CoreDNS configuration to implement a custom domain rewrite rule. We want to rewrite queries for `example.local` to `custom-app.apps.svc.cluster.local`:

```bash
kubectl edit configmap coredns -n kube-system
```

In the editor, find the Corefile section and add the rewrite rule before the forward plugin:

```
rewrite name example.local custom-app.apps.svc.cluster.local
```

After modification, the Corefile might look something like this:

```
Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        rewrite name example.local custom-app.apps.svc.cluster.local
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

Save the changes. Now, we need to restart the CoreDNS pods to apply the new configuration:

```bash
kubectl delete pod -l k8s-app=kube-dns -n kube-system
```

The CoreDNS pods will be recreated automatically with the new configuration.

Finally, let's verify that the DNS changes are working correctly:

```bash
kubectl exec -it dns-client -- nslookup example.local
```

This should resolve to the same IP address as `custom-app.apps.svc.cluster.local`.

## Question 14: Node Troubleshooting

**Task:**

1. Investigate why the node is not ready
2. Check node events, conditions, and logs
3. Check the status of the kubelet service
4. Resolve any issues with the node (might be kubelet configuration, certificates, networking)
5. Verify the node returns to `Ready` state

**Solution:**

First, let's check the status of the node:

```bash
kubectl get nodes
```

We should see that `node-2` is in `NotReady` state. Let's investigate why:

```bash
kubectl describe node node-2
```

Look at the `Conditions` section to understand why the node is not ready. Common conditions include:

- `MemoryPressure`
- `DiskPressure`
- `PIDPressure`
- `Ready`

Let's check the events related to this node:

```bash
kubectl get events --field-selector involvedObject.name=node-2
```

Now, let's SSH into the node to investigate further:

```bash
ssh admin@node-2
```

Once on the node, check the status of the kubelet service:

```bash
sudo systemctl status kubelet
```

If the kubelet service is not running, start it:

```bash
sudo systemctl start kubelet
sudo systemctl enable kubelet
```

Check the kubelet logs for any errors:

```bash
sudo journalctl -u kubelet -n 100
```

Look for error messages like:

- Certificate issues
- API server connectivity issues
- Configuration errors
- Resource constraints

Let's check if the node can reach the API server:

```bash
curl -k https://<master-node-ip>:6443
```

If there are certificate issues, we might need to regenerate them:

```bash
sudo kubeadm certs renew all
```

Or we might need to reset the node:

```bash
sudo kubeadm reset
```

And then rejoin it to the cluster:

```bash
sudo kubeadm join <master-node-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

If the issue is with network connectivity, check the CNI configuration:

```bash
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/*
```

Make sure that the required CNI plugins are installed and configured correctly.

If disk pressure is the issue, clean up the node:

```bash
sudo docker system prune -af  # For Docker runtime
sudo crictl rmi --prune       # For CRI-compatible runtimes
```

After resolving the issues, verify that the node is now in `Ready` state:

```bash
kubectl get nodes
```

## Question 15: Cluster Component Troubleshooting

**Task:**

1. Check the status and logs of the kube-apiserver
2. Identify any errors or warnings in the logs
3. Check the status of etcd and its connectivity with kube-apiserver
4. Resolve any detected issues
5. Verify the kube-apiserver is functioning properly

**Solution:**

First, let's check the status of the kube-apiserver:

```bash
kubectl get pods -n kube-system | grep kube-apiserver
```

If the kube-apiserver is running as a static pod (common in kubeadm-based clusters), its name will be something like `kube-apiserver-<master-node-name>`.

Let's check the logs of the kube-apiserver:

```bash
kubectl logs kube-apiserver-<master-node-name> -n kube-system
```

Look for error messages or warnings in the logs. These can help identify issues with the kube-apiserver.

Now, let's check the status of etcd:

```bash
kubectl get pods -n kube-system | grep etcd
kubectl logs etcd-<master-node-name> -n kube-system
```

To check etcd's connectivity with the kube-apiserver, SSH into the master node:

```bash
ssh admin@<master-node-ip>
```

On the master node, check the kube-apiserver's configuration to see how it connects to etcd:

```bash
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd
```

You should see the etcd endpoints, as well as the paths to the certificates used to connect to etcd.

Let's check if etcd is healthy:

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

If there are issues with etcd, we might need to restore from a backup or fix configuration issues.

Common kube-apiserver issues include:

1. Certificate expiration:
    
    ```bash
    sudo kubeadm certs check-expiration
    sudo kubeadm certs renew all
    ```
    
2. Incorrect configuration:
    
    ```bash
    sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
    ```
    
    Fix any configuration errors and save the file. The kubelet will automatically restart the kube-apiserver pod.
    
3. Resource constraints:
    
    ```bash
    # Check if the master node has enough resources
    top
    df -h
    ```
    
4. Networking issues:
    
    ```bash
    # Check if the API server can bind to its port
    sudo netstat -tulpn | grep 6443
    ```
    

After resolving the issues, verify that the kube-apiserver is functioning properly:

```bash
# On the master node
sudo crictl ps | grep kube-apiserver

# From kubectl
kubectl get nodes
kubectl api-resources
```

If these commands return without errors, the kube-apiserver is functioning properly.

## Question 16: Application Resource Monitoring

**Task:**

1. Deploy metrics-server in your cluster if not already present
2. Find the pod consuming the most CPU in the `kube-system` namespace
3. Find the node with the highest memory utilization
4. Create a Pod that has resource requests and limits set appropriately
5. Configure a HorizontalPodAutoscaler for a deployment based on CPU usage

**Solution:**

First, let's deploy the metrics-server in our cluster if it's not already present:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Wait for the metrics-server pod to be ready:

```bash
kubectl wait --for=condition=Ready pod -l k8s-app=metrics-server -n kube-system
```

Now, let's find the pod consuming the most CPU in the `kube-system` namespace:

```bash
kubectl top pods -n kube-system --sort-by=cpu
```

This command will list all pods in the `kube-system` namespace, sorted by CPU usage.

Next, let's find the node with the highest memory utilization:

```bash
kubectl top nodes --sort-by=memory
```

This command will list all nodes, sorted by memory usage.

Now, let's create a Pod with appropriate resource requests and limits:

```yaml
# resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: resource-container
    image: nginx:latest
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
```

Apply the Pod:

```bash
kubectl apply -f resource-pod.yaml
```

Finally, let's create a Deployment and configure a HorizontalPodAutoscaler for it based on CPU usage:

```yaml
# autoscaling-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: autoscaling-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: autoscaling-app
  template:
    metadata:
      labels:
        app: autoscaling-app
    spec:
      containers:
      - name: autoscaling-container
        image: nginx:latest
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

Apply the Deployment:

```bash
kubectl apply -f autoscaling-deployment.yaml
```

Now, let's create the HorizontalPodAutoscaler:

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: autoscaling-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: autoscaling-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply the HorizontalPodAutoscaler:

```bash
kubectl apply -f hpa.yaml
```

Let's verify that everything is working correctly:

```bash
# Check the Pod with resource requests and limits
kubectl describe pod resource-pod

# Check the HorizontalPodAutoscaler
kubectl get hpa autoscaling-hpa
kubectl describe hpa autoscaling-hpa
```

The HorizontalPodAutoscaler will automatically scale the Deployment when the average CPU utilization across all Pods exceeds 50%.

## Question 17: Container Output Troubleshooting

**Task:**

1. Check the logs of the pod
2. Identify any error messages or patterns
3. Export the logs to a file for further analysis
4. Connect to the pod using `kubectl exec` and check internal application state
5. Fix the issue and verify the application starts working correctly

**Solution:**

First, let's check the logs of the pod:

```bash
kubectl logs logging-app
```

If the pod has multiple containers, we need to specify the container name:

```bash
kubectl logs logging-app -c <container-name>
```

If the pod has restarted, we can look at the logs of the previous instance:

```bash
kubectl logs logging-app --previous
```

Let's export the logs to a file for further analysis:

```bash
kubectl logs logging-app > logging-app.log
```

Now, let's connect to the pod using `kubectl exec` and check the internal application state:

```bash
kubectl exec -it logging-app -- /bin/sh
```

Inside the pod, we can check various aspects of the application:

```bash
# Check the environment variables
env

# Check the file system
ls -la

# Check the processes
ps aux

# Check the network connectivity
netstat -tuln

# Check the application configuration
cat /app/config.json  # Replace with the actual configuration file path

# Check the application log files
cat /app/logs/app.log  # Replace with the actual log file path
```

Based on our investigation, we can identify the issue and fix it. Common issues include:

1. Configuration errors:
    
    ```bash
    # Create a ConfigMap with the correct configuration
    kubectl create configmap app-config --from-file=config.json
    
    # Update the pod to use the ConfigMap
    kubectl edit pod logging-app
    ```
    
2. Environment variable issues:
    
    ```bash
    # Update the environment variables
    kubectl edit pod logging-app
    ```
    
3. Permission issues:
    
    ```bash
    # Update the security context
    kubectl edit pod logging-app
    ```
    
4. Resource constraints:
    
    ```bash
    # Update the resource requests and limits
    kubectl edit pod logging-app
    ```
    
5. Application code issues:
    
    ```bash
    # Update the image to a fixed version
    kubectl edit pod logging-app
    ```
    

After fixing the issue, let's verify that the application starts working correctly:

```bash
# Check the pod status
kubectl get pod logging-app

# Check the logs again
kubectl logs logging-app

# Test the application
kubectl exec -it logging-app -- curl localhost
```

## Question 18: Network Troubleshooting

**Task:**

1. Verify that the CoreDNS pods are running correctly
2. Test connectivity between pods in different namespaces
3. Check if kube-proxy is running on all nodes
4. Verify that service endpoints are correctly configured
5. Troubleshoot and fix any service discovery or connectivity issues you find

**Solution:**

First, let's verify that the CoreDNS pods are running correctly:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

The CoreDNS pods should be in the `Running` state with status `Ready: 1/1`. If they're not, check the logs:

```bash
kubectl logs -l k8s-app=kube-dns -n kube-system
```

Now, let's test connectivity between pods in different namespaces. First, let's create test pods in two different namespaces:

```bash
# Create namespaces
kubectl create namespace test-ns1
kubectl create namespace test-ns2

# Create a pod in test-ns1
kubectl run test-pod1 -n test-ns1 --image=nginx

# Create a pod in test-ns2
kubectl run test-pod2 -n test-ns2 --image=nginx
```

Let's expose these pods as services:

```bash
kubectl expose pod test-pod1 -n test-ns1 --port=80
kubectl expose pod test-pod2 -n test-ns2 --port=80
```

Now, let's test connectivity from test-pod1 to test-pod2:

```bash
kubectl exec -it test-pod1 -n test-ns1 -- curl test-pod2.test-ns2.svc.cluster.local
```

Next, let's check if kube-proxy is running on all nodes:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

There should be one kube-proxy pod running on each node. If any are missing or not in the `Running` state, check their logs:

```bash
kubectl logs -l k8s-app=kube-proxy -n kube-system
```

Now, let's verify that service endpoints are correctly configured. Let's check the endpoints for our test services:

```bash
kubectl get endpoints test-pod1 -n test-ns1
kubectl get endpoints test-pod2 -n test-ns2
```

Each endpoint should have the IP address of the corresponding pod. If not, there's an issue with service-to-pod mapping.

Let's check for NetworkPolicies that might be blocking traffic:

```bash
kubectl get networkpolicies --all-namespaces
```

If there are NetworkPolicies, make sure they allow the necessary traffic:

```bash
kubectl describe networkpolicy -n test-ns1
kubectl describe networkpolicy -n test-ns2
```

If there are issues with service discovery or connectivity, here are some common solutions:

1. CoreDNS issues:
    
    ```bash
    # Check the CoreDNS configuration
    kubectl get configmap coredns -n kube-system -o yaml
    
    # Restart the CoreDNS pods
    kubectl delete pod -l k8s-app=kube-dns -n kube-system
    ```
    
2. kube-proxy issues:
    
    ```bash
    # Restart kube-proxy
    kubectl delete pod -l k8s-app=kube-proxy -n kube-system
    ```
    
3. NetworkPolicy issues:
    
    ```bash
    # Create an allow-all NetworkPolicy
    kubectl apply -f - <<EOF
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: allow-all
      namespace: test-ns1
    spec:
      podSelector: {}
      ingress:
      - {}
      egress:
      - {}
      policyTypes:
      - Ingress
      - Egress
    EOF
    ```
    
4. Service endpoint issues:
    
    ```bash
    # Delete and recreate the service
    kubectl delete service test-pod1 -n test-ns1
    kubectl expose pod test-pod1 -n test-ns1 --port=80
    ```
    

After fixing any issues, verify that connectivity is working correctly:

```bash
kubectl exec -it test-pod1 -n test-ns1 -- curl test-pod2.test-ns2.svc.cluster.local
```

## Question 19: Multi-Component Troubleshooting

**Task:**

1. An application deployment named `web-app` in namespace `frontend` is not scaling properly - investigate and fix the issue
2. PersistentVolumeClaims in the `database` namespace are stuck in `Pending` state - identify and resolve the problem
3. Pods in the `monitoring` namespace can't reach services in other namespaces - fix the network connectivity issue
4. The `kube-scheduler` pod appears to be crashing repeatedly - investigate and resolve the issue

**Solution:**

### 1. Fix the web-app scaling issue:

First, let's check the deployment:

```bash
kubectl get deployment web-app -n frontend
kubectl describe deployment web-app -n frontend
```

Let's check if there's a HorizontalPodAutoscaler configured:

```bash
kubectl get hpa -n frontend
kubectl describe hpa -n frontend
```

Let's check the ReplicaSet and Pods:

```bash
kubectl get rs -n frontend
kubectl get pods -n frontend
kubectl describe pods -n frontend -l app=web-app
```

Common issues with scaling include:

1. Resource constraints at the namespace level:
    
    ```bash
    kubectl get resourcequota -n frontend
    kubectl describe resourcequota -n frontend
    ```
    
2. Resource constraints at the node level:
    
    ```bash
    kubectl describe nodes
    ```
    
3. HPA configuration issues:
    
    ```bash
    # Fix the HPA or create it if it doesn't exist
    kubectl apply -f - <<EOF
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: web-app-hpa
      namespace: frontend
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: web-app
      minReplicas: 1
      maxReplicas: 10
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    EOF
    ```
    
4. Pod startup issues:
    
    ```bash
    # Check pod events
    kubectl get events -n frontend --sort-by='.lastTimestamp'
    ```
    

### 2. Fix PersistentVolumeClaims issues:

Let's check the PVCs in the database namespace:

```bash
kubectl get pvc -n database
kubectl describe pvc -n database
```

Let's check the StorageClass being used:

```bash
kubectl get storageclass
kubectl describe storageclass
```

Check for available PersistentVolumes:

```bash
kubectl get pv
```

Common issues with PVCs include:

1. No StorageClass:
    
    ```bash
    # Create a StorageClass
    kubectl apply -f - <<EOF
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: standard
    provisioner: kubernetes.io/gce-pd  # Change this to your cloud provider's provisioner
    parameters:
      type: pd-standard
    reclaimPolicy: Delete
    allowVolumeExpansion: true
    volumeBindingMode: Immediate
    EOF
    ```
    
2. Storage provisioner issues:
    
    ```bash
    # Check if the storage provisioner is running
    kubectl get pods -n kube-system | grep -i provisioner
    ```
    
3. PV and PVC mismatch:
    
    ```bash
    # Create a PV that matches the PVC
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-database
    spec:
      capacity:
        storage: 10Gi
      accessModes:
      - ReadWriteOnce
      persistentVolumeReclaimPolicy: Delete
      storageClassName: standard
      hostPath:
        path: /mnt/data
    EOF
    ```
    

### 3. Fix network connectivity issue:

Let's check for NetworkPolicies that might be blocking traffic:

```bash
kubectl get networkpolicy -n monitoring
kubectl describe networkpolicy -n monitoring
```

Let's test connectivity from a pod in the monitoring namespace to a service in another namespace:

```bash
# Create a test pod in the monitoring namespace
kubectl run test-pod -n monitoring --image=busybox -- sleep 3600

# Test connectivity to a service in another namespace
kubectl exec -it test-pod -n monitoring -- wget -O- <service-name>.<namespace>.svc.cluster.local
```

Common issues with network connectivity include:

1. Restrictive NetworkPolicies:
    
    ```bash
    # Create an allow-all NetworkPolicy
    kubectl apply -f - <<EOF
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: allow-all
      namespace: monitoring
    spec:
      podSelector: {}
      ingress:
      - {}
      egress:
      - {}
      policyTypes:
      - Ingress
      - Egress
    EOF
    ```
    
2. DNS issues:
    
    ```bash
    # Check DNS resolution
    kubectl exec -it test-pod -n monitoring -- nslookup kubernetes.default.svc.cluster.local
    ```
    
3. kube-proxy issues:
    
    ```bash
    # Check kube-proxy pods
    kubectl get pods -n kube-system -l k8s-app=kube-proxy
    ```
    

### 4. Fix kube-scheduler issues:

Let's check the status of the kube-scheduler pod:

```bash
kubectl get pods -n kube-system -l component=kube-scheduler
kubectl describe pod -n kube-system -l component=kube-scheduler
```

Let's check the logs:

```bash
kubectl logs -n kube-system -l component=kube-scheduler
```

Common issues with the kube-scheduler include:

1. Configuration issues:
    
    ```bash
    # SSH into the control plane node
    ssh admin@<control-plane-node>
    
    # Check the kube-scheduler manifest
    sudo cat /etc/kubernetes/manifests/kube-scheduler.yaml
    
    # Fix any configuration issues
    sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml
    ```
    
2. Certificate issues:
    
    ```bash
    # Renew certificates
    sudo kubeadm certs renew all
    ```
    
3. Resource constraints:
    
    ```bash
    # Check resource usage
    top
    df -h
    ```
    

After fixing all these issues, verify that everything is working correctly:

```bash
# Check the deployment scaling
kubectl get deployment web-app -n frontend

# Check the PVCs
kubectl get pvc -n database

# Check network connectivity
kubectl exec -it test-pod -n monitoring -- wget -O- <service-name>.<namespace>.svc.cluster.local

# Check the kube-scheduler
kubectl get pods -n kube-system -l component=kube-scheduler
```

# Question 20: Resource Management and Quota Troubleshooting

## Task Overview

1. Check the resource quotas in the `restricted` namespace
2. Identify why a deployment named `resource-hungry-app` is not scheduling pods
3. Modify the deployment to fit within the namespace resource constraints
4. Implement an appropriate LimitRange for the namespace
5. Verify the deployment is now functioning correctly within the constraints

## Detailed Solution

### Step 1: Investigating the Resource Quotas

First, we need to examine what resource quotas exist in the `restricted` namespace:

```bash
kubectl get resourcequota -n restricted
```

This command would show something like:

```
NAME               AGE   REQUEST                                                    USED   HARD
restricted-quota   2d    cpu: 800m/1, memory: 800Mi/1Gi, pods: 3/5
```

To get more details about the quota:

```bash
kubectl describe resourcequota restricted-quota -n restricted
```

Output might look like:

```
Name:            restricted-quota
Namespace:       restricted
Resource         Used    Hard
--------         ----    ----
cpu              800m    1
memory           800Mi   1Gi
pods             3       5
```

This tells us that in the `restricted` namespace:

- CPU is limited to 1 core, with 800m already used
- Memory is limited to 1Gi, with 800Mi already used
- Pods are limited to 5, with 3 already running

### Step 2: Identifying Deployment Issues

Let's examine the deployment that's having problems:

```bash
kubectl get deployment resource-hungry-app -n restricted
```

We might see:

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
resource-hungry-app   0/3     0            0           1h
```

This shows the deployment is trying to create 3 replicas, but none are available.

To investigate further:

```bash
kubectl describe deployment resource-hungry-app -n restricted
```

In the events section, we might see messages like:

```
  Type     Reason             Age   From                   Message
  ----     ------             ----  ----                   -------
  Warning  FailedCreate       5m    replicaset-controller  Error creating: pods "resource-hungry-app-65d8fb7b54-2xvjp" is forbidden: exceeded quota: restricted-quota, requested: cpu=500m,memory=512Mi, used: cpu=800m,memory=800Mi, limited: cpu=1,memory=1Gi
```

To see the current resource requests of the deployment:

```bash
kubectl get deployment resource-hungry-app -n restricted -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

This might show:

```json
{"limits":{"cpu":"500m","memory":"512Mi"},"requests":{"cpu":"500m","memory":"512Mi"}}
```

From these findings, we understand that:

1. Each pod is requesting 500m CPU and 512Mi memory
2. The deployment wants to create 3 replicas (1.5 CPU and 1536Mi total)
3. The namespace only has 200m CPU and 224Mi memory remaining
4. This is why new pods cannot be scheduled

### Step 3: Modifying the Deployment

We need to adjust the deployment to fit within the available resources. Two approaches:

1. Reduce the resource requests per pod:

```bash
kubectl patch deployment resource-hungry-app -n restricted --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/cpu", "value":"150m"},
       {"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value":"200Mi"},
       {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/cpu", "value":"100m"},
       {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/memory", "value":"128Mi"}]'
```

2. Reduce the number of replicas:

```bash
kubectl scale deployment resource-hungry-app -n restricted --replicas=1
```

For this solution, we'll do both:

```yaml
# updated-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-hungry-app
  namespace: restricted
spec:
  replicas: 2
  selector:
    matchLabels:
      app: resource-hungry-app
  template:
    metadata:
      labels:
        app: resource-hungry-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        resources:
          requests:
            memory: "100Mi"
            cpu: "100m"
          limits:
            memory: "200Mi"
            cpu: "150m"
```

Apply the updated deployment:

```bash
kubectl apply -f updated-deployment.yaml
```

### Step 4: Implementing a LimitRange

To ensure future deployments follow resource constraints, let's create a LimitRange:

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: restricted-limits
  namespace: restricted
spec:
  limits:
  - type: Container
    default:
      cpu: "150m"
      memory: "200Mi"
    defaultRequest:
      cpu: "100m"
      memory: "100Mi"
    max:
      cpu: "300m"
      memory: "300Mi"
    min:
      cpu: "50m"
      memory: "50Mi"
  - type: Pod
    max:
      cpu: "600m"
      memory: "600Mi"
```

This LimitRange does several things:

- Sets default resource limits for containers that don't specify them
- Sets default resource requests for containers that don't specify them
- Sets maximum and minimum resource limits for containers
- Sets maximum resource limits for entire pods

Apply the LimitRange:

```bash
kubectl apply -f limit-range.yaml
```

### Step 5: Verifying the Solution

Now, let's verify that our changes worked:

```bash
kubectl get deployment resource-hungry-app -n restricted
```

We should see:

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
resource-hungry-app   2/2     2            2           1h30m
```

Check the pods:

```bash
kubectl get pods -n restricted -l app=resource-hungry-app
```

Output should show all pods are running:

```
NAME                                   READY   STATUS    RESTARTS   AGE
resource-hungry-app-86d9fb7b54-2j4qp   1/1     Running   0          2m
resource-hungry-app-86d9fb7b54-8k3lp   1/1     Running   0          2m
```

Verify the LimitRange is active:

```bash
kubectl describe limitrange restricted-limits -n restricted
```

Check how much of the quota is being used now:

```bash
kubectl describe resourcequota restricted-quota -n restricted
```

We should see something like:

```
Name:            restricted-quota
Namespace:       restricted
Resource         Used    Hard
--------         ----    ----
cpu              1000m   1
memory           960Mi   1Gi
pods             5       5
```

Let's test creating a new pod without resource specifications:

```yaml
# test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: restricted
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

Apply the test pod:

```bash
kubectl apply -f test-pod.yaml
```

Then check its resource allocations:

```bash
kubectl describe pod test-pod -n restricted
```

The output should show that the LimitRange default values were applied:

```
    Limits:
      cpu:     150m
      memory:  200Mi
    Requests:
      cpu:        100m
      memory:     100Mi
```

## Conclusion

We've successfully identified and resolved the resource quota issues:
