# cka-study

## **Certified Kubernetes Administrator (CKA) Study Guide**

Welcome to the **CKA Study Guide**. This guide provides a structured breakdown of Kubernetes concepts and practical labs to help you prepare for the Certified Kubernetes Administrator (CKA) exam. The content is organized into chapters and branches to align with specific exam topics.

---

## **Structure of the Repository**

The repository is divided into branches, each covering a specific topic from the CKA syllabus. Each branch contains:

- **Docs Folder:** Contains study notes and reference materials.
    
- **Labs Folder:** Includes hands-on lab scenarios and configurations.
    

---

## **Groupings by Topic**

### **Part I: Kubernetes Fundamentals**

#### **Branch 1: Introduction to Kubernetes**

Topics include:

- Velocity and immutability
    
- Declarative configuration
    
- Self-healing systems
    
- Cloud-native ecosystem
    

🔧 **Lab:**

- Set up a Kubernetes cluster using Minikube or k3s.
    
- Practice basic `kubectl` commands.
    

#### **Branch 2: Kubernetes Architecture**

Topics include:

- Kubernetes components (API server, etcd, scheduler, kubelet, kube-proxy)
    
- Abstracting infrastructure
    
- Scaling clusters
    

🔧 **Lab:**

- Explore cluster components using `kubectl`.
    

---

### **Part II: Creating and Running Containers**

#### **Branch 3: Container Images and Dockerfiles**

Topics include:

- Building application images
    
- Optimizing image sizes
    
- Multistage builds
    

🔧 **Lab:**

- Create a Dockerfile and push an image to a registry.
    
- Deploy a container with resource limits.
    

#### **Branch 4: Container Runtime Interface (CRI)**

Topics include:

- Running containers with Docker
    
- Exploring the kuard application
    

🔧 **Lab:**

- Deploy a sample application and inspect running Pods.
    

---

### **Part III: Deploying Kubernetes Clusters**

#### **Branch 5: Cluster Setup**

Topics include:

- Installing Kubernetes on public cloud providers (GKE, AKS, EKS)
    
- Running Kubernetes locally (Minikube)
    

🔧 **Lab:**

- Deploy a cluster on GKE and verify using `kubectl`.
    

#### **Branch 6: Kubernetes Proxy and DNS**

Topics include:

- Kubernetes DNS
    
- Cluster networking
    

🔧 **Lab:**

- Explore DNS configurations within the cluster.
    

---

### **Part IV: Common kubectl Commands**

#### **Branch 7: Namespaces and Contexts**

Topics include:

- Creating and managing namespaces
    
- Switching contexts
    

🔧 **Lab:**

- Practice creating namespaces and switching contexts.
    

#### **Branch 8: Debugging Commands**

Topics include:

- Viewing logs
    
- Running commands inside containers
    

🔧 **Lab:**

- Use `kubectl logs` and `kubectl exec` for troubleshooting.
    

---

### **Part V: Pods and Deployments**

#### **Branch 9: Pods**

Topics include:

- Creating and managing Pods
    
- Health checks (liveness, readiness, startup probes)
    

🔧 **Lab:**

- Create a Pod manifest and implement health probes.
    

#### **Branch 10: Deployments**

Topics include:

- Managing Deployments
    
- Rolling updates and rollout history
    

🔧 **Lab:**

- Deploy an application and perform a rolling update.
    

---

### **Part VI: Services and Ingress**

#### **Branch 11: Services**

Topics include:

- ClusterIP, NodePort, and LoadBalancer
    
- Service discovery
    

🔧 **Lab:**

- Expose an application using different service types.
    

#### **Branch 12: Ingress**

Topics include:

- Ingress controllers
    
- TLS and advanced Ingress topics
    

🔧 **Lab:**

- Set up an Ingress controller and configure rules.
    

---

### **Part VII: ConfigMaps, Secrets, and Jobs**

#### **Branch 13: ConfigMaps and Secrets**

Topics include:

- Creating and managing ConfigMaps
    
- Handling Secrets securely
    

🔧 **Lab:**

- Use ConfigMaps and Secrets in a Pod.
    

#### **Branch 14: Jobs and CronJobs**

Topics include:

- One-shot and parallel Jobs
    
- Scheduling tasks with CronJobs
    

🔧 **Lab:**

- Create Jobs and CronJobs for scheduled tasks.
    

---

### **Part VIII: Security and Networking**

#### **Branch 15: Role-Based Access Control (RBAC)**

Topics include:

- Roles, RoleBindings, and ClusterRoles
    
- Managing RBAC policies
    

🔧 **Lab:**

- Create and apply RBAC policies.
    

#### **Branch 16: Network Policies**

Topics include:

- Securing Pod communication
    
- Implementing Network Policies
    

🔧 **Lab:**

- Apply Network Policies to restrict traffic.
    

---

### **Part IX: Troubleshooting and Storage**

#### **Branch 17: Troubleshooting Kubernetes Clusters**

Topics include:

- Debugging Pods and nodes
    
- Identifying common issues
    

🔧 **Lab:**

- Simulate cluster issues and troubleshoot.
    

#### **Branch 18: Storage Solutions**

Topics include:

- PersistentVolumes and PersistentVolumeClaims
    
- Dynamic volume provisioning
    

🔧 **Lab:**

- Create and use PersistentVolumes in a Pod.
    

---

### **Part X: Extending Kubernetes and Multi-Cluster Deployments**

#### **Branch 19: Custom Resources and Operators**

Topics include:

- CustomResourceDefinitions (CRDs)
    
- Operators
    

🔧 **Lab:**

- Create a CRD and deploy an Operator.
    

#### **Branch 20: Multi-Cluster Deployments**

Topics include:

- Deploying applications across multiple clusters
    

🔧 **Lab:**

- Set up a multi-cluster environment and deploy an application.
    

---

### **Final Preparation**

#### **Branch 21: Mock Exams and Practice**

Topics include:

- Using Killer.sh simulator
    
- Completing mock exams
    

🔧 **Lab:**

- Take practice exams and review weak areas.
    

---

## **Tools and Lab Environment**

To simulate the concepts, the following tools will be used:

- **Minikube**
    
- **k3s**
    
- **kubectl**
    
- **Docker**
    

## **Getting Started**

1. Clone the repository to your local machine.
    
2. Checkout the branch corresponding to the topic you want to study:
    

```
git checkout <branch-name>
```

3. Explore the **Docs** and **Labs** folders for materials and labs.
    
4. Follow the weekly study plan to progress through the topics systematically.
    

## **Contributing**

We welcome contributions to improve the guide. Follow these steps:

1. Fork the repository.
    
2. Create a new branch:
    

```
git checkout -b feature/YourFeature
```

3. Commit your changes:
    

```
git commit -m 'Add YourFeature'
```

4. Push to your branch:
    

```
git push origin feature/YourFeature
```

5. Open a Pull Request.