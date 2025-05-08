# Kubernetes Networking & Policies: Cluster-Wide and Namespace-Scoped

## Introduction to Kubernetes Networking

Kubernetes networking enables communication between different components within a Kubernetes cluster. Understanding networking in Kubernetes is essential for building secure, efficient, and well-structured applications.

## Core Networking Concepts

### Pod Networking

At the foundation of Kubernetes networking is the Pod. Every Pod in Kubernetes receives its own unique IP address. This fundamental design principle enables Pods to communicate with each other without complex NAT configurations.

For example, if Pod A (10.244.1.4) wants to communicate with Pod B (10.244.2.5), it can do so directly using Pod B's IP address, regardless of which nodes they are running on.

```yaml
# Example of a simple Pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

### Service Types

Services are the primary way to expose applications running in Pods. Kubernetes offers several service types to accommodate different networking requirements:

#### ClusterIP Services (Internal Communication)

ClusterIP services provide internal cluster communication. They're accessible only within the cluster and are the default service type.

```yaml
# ClusterIP Service Example
apiVersion: v1
kind: Service
metadata:
  name: my-internal-service
  namespace: app-namespace
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP  # This is default and can be omitted
```

The service creates a stable internal IP address and DNS name (`my-internal-service.app-namespace.svc.cluster.local`) that other Pods can use to communicate with your application, regardless of which specific Pods are handling the requests.

#### NodePort Services (External Access)

NodePort services expose your application on a static port on each node in the cluster, making it accessible from outside the cluster.

```yaml
# NodePort Service Example
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
  namespace: app-namespace
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30007  # Must be in range 30000-32767
  type: NodePort
```

With this configuration, your application becomes accessible through any node's IP address on port 30007 (e.g., `http://<node-ip>:30007`).

#### LoadBalancer Services (Cloud Provider Integration)

LoadBalancer services provision an external load balancer from your cloud provider to route traffic to your service.

```yaml
# LoadBalancer Service Example
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer
  namespace: production
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

This creates an external load balancer with a public IP address that forwards traffic to your service.

## Network Policies

Network Policies act as firewalls within Kubernetes, controlling traffic flow between Pods. Let's explore both namespace-scoped and cluster-wide network policies.

### Namespace-Scoped Network Policies

Namespace-scoped policies apply only to Pods within a specific namespace. They're ideal for implementing isolation between different applications or environments.

```yaml
# Namespace-Scoped Network Policy Example
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal-traffic
  namespace: app-namespace  # This policy applies only in this namespace
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

This policy allows only Pods with label `app: frontend` in the `app-namespace` to communicate with Pods labeled `app: backend` on port 8080.

### Cluster-Wide Network Policies

While Kubernetes' native NetworkPolicy resource is namespace-scoped, you can create cluster-wide network policies using CustomResourceDefinitions (CRDs) provided by network plugin solutions like Calico, Cilium, or Antrea.

Here's an example using Calico's GlobalNetworkPolicy:

```yaml
# Cluster-Wide Network Policy with Calico
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny-all
spec:
  selector: all()  # Applies to all workloads in all namespaces
  types:
  - Ingress
  - Egress
```

This policy blocks all traffic to and from all Pods across all namespaces, creating a default-deny posture for the entire cluster. You would then define more specific policies to allow required communication.

### Practical Examples

#### Example 1: Namespace Isolation

Isolate the `production` namespace from the `development` namespace:

```yaml
# Deny all ingress from development to production
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-development
  namespace: production
spec:
  podSelector: {}  # Applies to all pods in the production namespace
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["development"]
```

#### Example 2: Allow DB Access Only to Backend

Restrict database access to only backend Pods:

```yaml
# Allow only backend pods to access the database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-access-policy
  namespace: data
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
      namespaceSelector:
        matchLabels:
          purpose: application
    ports:
    - protocol: TCP
      port: 5432
```

#### Example 3: Cluster-Wide DNS Access with Calico

Allow all Pods cluster-wide to access DNS services:

```yaml
# Cluster-wide access to DNS
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-dns-access
spec:
  selector: all()
  egress:
  - action: Allow
    protocol: UDP
    destination:
      selector: k8s-app == "kube-dns"
      ports:
      - 53
```

## Best Practices for Network Policies

1. **Start with deny-all**: Implement default deny policies and then explicitly allow necessary traffic.
2. **Namespace isolation**: Use network policies to isolate different environments (e.g., development, staging, production).
3. **Least privilege**: Restrict communication to only what's required for your applications to function.
4. **Documentation**: Maintain clear documentation of your network policies and their intended purpose.
5. **Regular audits**: Periodically review and update network policies to ensure they align with current requirements.

## Implementing Network Policies: Step-by-Step Guide

### 1. Ensure Network Policy Support

First, verify that your Kubernetes cluster supports Network Policies. Not all Kubernetes network plugins support Network Policies.

```bash
# Check if your network plugin supports Network Policies
kubectl get pods -n kube-system
```

Look for network plugins like Calico, Cilium, Weave Net, or Antrea which support Network Policies.

### 2. Create Default Deny Policy

Start with a default deny policy in your namespaces to ensure all traffic is blocked unless explicitly allowed:

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: your-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Apply this policy:
```bash
kubectl apply -f default-deny.yaml
```

### 3. Create Specific Allow Policies

Next, create policies to allow specific traffic:

```yaml
# allow-app-traffic.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-traffic
  namespace: your-namespace
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 80
```

Apply this policy:
```bash
kubectl apply -f allow-app-traffic.yaml
```

### 4. Test Your Network Policies

Test your policies by attempting to communicate between Pods:

```bash
# Start a test pod in your namespace
kubectl run test-pod --image=busybox -n your-namespace -- sleep 3600

# Try to connect to a service
kubectl exec -it test-pod -n your-namespace -- wget -O- http://web-service
```

If your policies are configured correctly, this attempt should succeed or fail according to your policy rules.

## Conclusion

Kubernetes networking provides a flexible foundation for building complex, distributed applications. Network Policies, whether namespace-scoped or cluster-wide, add essential security controls to restrict traffic flow within your cluster.

By understanding and implementing appropriate network policies, you can enhance the security posture of your Kubernetes applications and ensure that only authorized communication paths exist between your services.