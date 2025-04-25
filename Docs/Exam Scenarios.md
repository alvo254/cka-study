
# Certified Kubernetes Administrator (CKA) Exam Scenario Questions

## Storage (10%)

### Question 1: Dynamic Volume Provisioning

You need to create a storage solution for a database that requires persistent storage. The storage should be dynamically provisioned when pods request it.

**Task:**

1. Create a StorageClass named `fast-storage` that uses the `standard` provisioner
2. Configure the StorageClass with `reclaimPolicy: Retain`
3. Create a PersistentVolumeClaim named `db-data` in the `database` namespace that requests 10Gi of storage using this StorageClass
4. Create a Pod named `db-pod` that uses this PersistentVolumeClaim mounted at `/var/lib/mysql`

### Question 2: Volume Management

There is an existing PersistentVolume named `legacy-data` in your cluster that needs to be repurposed.

**Task:**

1. Identify the current status of the PersistentVolume `legacy-data`
2. If it's bound to a claim, delete the claim (but not any associated pods)
3. Modify the PersistentVolume to change its reclaim policy to `Recycle`
4. Create a new PersistentVolumeClaim named `app-data` in the `default` namespace that can bind to this volume

## Workloads and Scheduling (15%)

### Question 3: Deployment Rollout and Rollback

An application deployment named `web-app` in the `production` namespace is experiencing issues after a recent update.

**Task:**

1. Check the rollout history of the deployment
2. Roll back the deployment to the previous stable version
3. Confirm the application is running with 3 replicas
4. Configure the deployment to use the `RollingUpdate` strategy with `maxSurge: 1` and `maxUnavailable: 0`

### Question 4: ConfigMaps and Secrets

You need to deploy an application that requires both configuration data and sensitive credentials.

**Task:**

1. Create a ConfigMap named `app-config` with the following data:
    - `DB_HOST=db.example.com`
    - `DB_PORT=5432`
    - `APP_MODE=production`
2. Create a Secret named `app-secrets` with the following data:
    - `DB_USER=admin`
    - `DB_PASSWORD=s3cret`
3. Create a Deployment named `backend-app` that mounts both the ConfigMap and Secret as environment variables

### Question 5: Pod Scheduling and Affinity

You need to ensure specific pods run on particular nodes in your cluster.

**Task:**

1. Label one of your worker nodes with `disk=ssd`
2. Create a Pod named `data-processor` that will be scheduled only on nodes with the `disk=ssd` label
3. Create another Pod named `web-frontend` that should preferably run on nodes with the `disk=ssd` label, but can run elsewhere if necessary
4. Create a Pod named `monitoring-agent` that must not be co-located with either of the above pods

## Cluster Architecture, Installation and Configuration (25%)

### Question 6: Kubeadm Cluster Setup

You need to initialize a new Kubernetes control plane node.

**Task:**

1. Create a kubeadm configuration file that:
    - Uses the `172.16.0.0/16` pod CIDR range
    - Sets the control plane endpoint to `k8s-master.example.com:6443`
    - Uses a specific version of Kubernetes (e.g., `v1.26.0`)
2. Initialize the control plane using this configuration
3. Configure kubectl for the admin user
4. Install a CNI plugin of your choice
5. Prepare a join command for worker nodes

### Question 7: RBAC Configuration

Your organization needs to implement role-based access control for different teams.

**Task:**

1. Create a new ServiceAccount named `developer-account` in the `development` namespace
2. Create a Role that allows the following permissions in the `development` namespace only:
    - Get, list, and watch pods
    - Create and delete deployments
    - Update and patch services
3. Bind this Role to the ServiceAccount
4. Create a ClusterRole that allows viewing resources across all namespaces
5. Bind this ClusterRole to the same ServiceAccount

### Question 8: Cluster Upgrades

You need to safely upgrade a Kubernetes cluster.

**Task:**

1. Check the current version of all components
2. Upgrade the kubeadm tool to the target version
3. Create a plan for upgrading the control plane node
4. Execute the upgrade of the control plane components
5. Update the kubelet and kubectl on the control plane node
6. Drain a worker node in preparation for its upgrade

### Question 9: Custom Resource Definitions

You need to extend your Kubernetes cluster with custom resources.

**Task:**

1. Create a Custom Resource Definition (CRD) named `backups.example.com` with:
    - Group: `example.com`
    - Version: `v1`
    - Scope: `Namespaced`
    - Validation schema requiring `schedule` and `destination` fields
2. Create a custom resource object of this type
3. Install a simple operator (such as the CertManager operator) using Helm

## Services and Networking (20%)

### Question 10: Network Policies

You need to implement network security for an application stack.

**Task:**

1. Create a namespace called `secure-apps`
2. Deploy a pod named `backend-api` with label `app=backend` in this namespace
3. Deploy a pod named `frontend-ui` with label `app=frontend` in this namespace
4. Create a NetworkPolicy named `api-policy` that:
    - Allows ingress traffic to `backend-api` only from `frontend-ui`
    - Allows egress traffic from `backend-api` only to pods with label `db=true`
    - Blocks all other traffic to and from the `backend-api`

### Question 11: Service Configuration

You need to expose applications to both internal and external users.

**Task:**

1. Create a Deployment named `web-server` with 3 replicas running nginx
2. Expose this deployment within the cluster using a ClusterIP service named `web-internal`
3. Create a NodePort service named `web-external` that exposes the deployment on port 30080
4. Create a LoadBalancer service named `web-lb` that exposes the deployment to external traffic
5. Verify connectivity to all three services

### Question 12: Gateway API Configuration

Implement the Gateway API to manage ingress traffic to your services.

**Task:**

1. Install the Gateway API CRDs in your cluster
2. Create a GatewayClass named `production-class`
3. Create a Gateway named `main-gateway` using this GatewayClass that listens on port 80
4. Create an HTTPRoute named `web-route` that routes traffic with the path `/app` to the `web-server` service
5. Verify the Gateway and HTTPRoute are properly configured

### Question 13: DNS Configuration

Configure DNS for service discovery in your cluster.

**Task:**

1. Inspect the CoreDNS configuration in your cluster
2. Create a service named `custom-app` in the `apps` namespace
3. Create a Pod that can resolve this service using DNS
4. Modify the CoreDNS configuration to implement a custom domain rewrite rule
5. Verify the DNS changes are working correctly

## Troubleshooting (30%)

### Question 14: Node Troubleshooting

A worker node named `node-2` is in `NotReady` state.

**Task:**

1. Investigate why the node is not ready
2. Check node events, conditions, and logs
3. Check the status of the kubelet service
4. Resolve any issues with the node (might be kubelet configuration, certificates, networking)
5. Verify the node returns to `Ready` state

### Question 15: Cluster Component Troubleshooting

Your kube-apiserver is experiencing intermittent failures.

**Task:**

1. Check the status and logs of the kube-apiserver
2. Identify any errors or warnings in the logs
3. Check the status of etcd and its connectivity with kube-apiserver
4. Resolve any detected issues
5. Verify the kube-apiserver is functioning properly

### Question 16: Application Resource Monitoring

You need to monitor resource usage in your cluster.

**Task:**

1. Deploy metrics-server in your cluster if not already present
2. Find the pod consuming the most CPU in the `kube-system` namespace
3. Find the node with the highest memory utilization
4. Create a Pod that has resource requests and limits set appropriately
5. Configure a HorizontalPodAutoscaler for a deployment based on CPU usage

### Question 17: Container Output Troubleshooting

An application pod named `logging-app` is not functioning correctly.

**Task:**

1. Check the logs of the pod
2. Identify any error messages or patterns
3. Export the logs to a file for further analysis
4. Connect to the pod using `kubectl exec` and check internal application state
5. Fix the issue and verify the application starts working correctly

### Question 18: Network Troubleshooting

Services in your cluster are having connectivity issues.

**Task:**

1. Verify that the CoreDNS pods are running correctly
2. Test connectivity between pods in different namespaces
3. Check if kube-proxy is running on all nodes
4. Verify that service endpoints are correctly configured
5. Troubleshoot and fix any service discovery or connectivity issues you find

### Question 19: Multi-Component Troubleshooting

Your cluster is experiencing issues with multiple components.

**Task:**

1. An application deployment named `web-app` in namespace `frontend` is not scaling properly - investigate and fix the issue
2. PersistentVolumeClaims in the `database` namespace are stuck in `Pending` state - identify and resolve the problem
3. Pods in the `monitoring` namespace can't reach services in other namespaces - fix the network connectivity issue
4. The `kube-scheduler` pod appears to be crashing repeatedly - investigate and resolve the issue

### Question 20: Resource Management and Quota Troubleshooting

A namespace has resource quotas that are preventing workloads from running.

**Task:**

1. Check the resource quotas in the `restricted` namespace
2. Identify why a deployment named `resource-hungry-app` is not scheduling pods
3. Modify the deployment to fit within the namespace resource constraints
4. Implement an appropriate LimitRange for the namespace
5. Verify the deployment is now functioning correctly within the constraints