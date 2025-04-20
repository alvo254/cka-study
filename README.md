# Certified Kubernetes Administrator (CKA) Study Guide

Welcome to the CKA Study Guide. This guide provides a detailed breakdown of the Kubernetes topics covered in the CKA exam, including exam tips, command-line tricks, and practical labs to reinforce your understanding. The content is structured into chapters and branches to align with specific exam objectives.

---


helm install cilium cilium/cilium --version 1.17.1 \
      --namespace kube-system \
      --set ipam.mode=kubernetes \
      --set kubeProxyReplacement=true \
      --set gatewayAPI.enabled=true \
      --set routingMode=tunnel \
      --set bpf.masquerade=true \
      --set prometheus.enabled=true \
      --set operator.prometheus.enabled=true \
      --set hubble.enabled=true \
      --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}" \
      --set hubble.relay.enabled=true \
      --set hubble.ui.enabled=true \
      --set nodePort.enabled=true \
      --set authentication.mutual.spire.enabled=true \
      --set authentication.mutual.spire.install.enabled=true \
      --set loadBalancer.enabled=true \
      --set loadBalancer.algorithm=maglev \
      --set ingressController.enabled=true \
      --set ingressController.default=true \
      --set ingressController.service.type=NodePort \
      --set ingressController.loadbalancerMode=shared \
      --set crds.install=true \
      --set tetragon.enabled=true \
      --set tetragon.export.hubble.enabled=true 




## Structure of the Repository

The repository is divided into branches, each focusing on a particular exam topic. Each branch contains:

- **Docs Folder:** Study notes and reference materials.
    
- **Labs Folder:** Hands-on lab exercises and configurations.
    

---

## Groupings by Topic

### **Part I: Exam Overview and Preparation**

#### **Branch 1: Exam Details and Resources**

**Topics include:**

- Exam Objectives
    
- Curriculum Overview
    
- Cluster Architecture, Installation, and Configuration
    
- Workloads and Scheduling
    
- Services and Networking
    
- Storage
    
- Troubleshooting
    
- Kubernetes Primitives Involved
    
- Exam Environment and Tips
    

🔧 **Lab:**

- Practice time management and command-line tips using kubectl.
    
- Set up context and namespace for efficient cluster management.
    

#### **Branch 2: Command-Line Tips and Tricks**

**Topics include:**

- Setting a Context and Namespace
    
- Using an Alias for kubectl
    
- Using kubectl Command Auto-Completion
    
- Internalizing Resource Short Names
    
- Deleting Kubernetes Objects
    
- Finding Object Information
    
- Discovering Command Options
    

🔧 **Lab:**

- Practice common kubectl commands.
    
- Explore kubectl aliases and auto-completion.
    

---

### **Part II: Cluster Architecture and Configuration**

#### **Branch 3: Cluster Architecture, Installation, and Configuration**

**Topics include:**

- Role-Based Access Control (RBAC)
    
- Creating and Managing Kubernetes Clusters
    
- Installing a Cluster
    
- Managing a Highly Available Cluster
    
- Upgrading a Cluster Version
    
- Backing Up and Restoring etcd
    

🔧 **Lab:**

- Install a Kubernetes cluster using kubeadm.
    
- Perform a cluster upgrade and etcd backup/restore.
    

---

### **Part III: Workloads and Scheduling**

#### **Branch 4: Workloads**

**Topics include:**

- Managing Workloads with Deployments
    
- Performing Rolling Updates and Rollbacks
    
- Scaling Workloads
    
- Defining and Consuming Configuration Data
    

🔧 **Lab:**

- Create and manage Deployments.
    
- Implement rolling updates and horizontal scaling.
    
- Use ConfigMaps and Secrets in a Pod.
    

---

### **Part IV: Scheduling and Tooling**

#### **Branch 5: Scheduling and Tooling**

**Topics include:**

- Defining Container Resource Requests and Limits
    
- Declarative Object Management Using Configuration Files
    
- Using Common Templating Tools (Helm, Kustomize, yq)
    

🔧 **Lab:**

- Practice setting resource requests/limits.
    
- Manage Kubernetes objects using Helm and Kustomize.
    

---

### **Part V: Services and Networking**

#### **Branch 6: Services and Networking**

**Topics include:**

- Kubernetes Networking Basics
    
- Connectivity Between Containers and Pods
    
- Understanding Services (ClusterIP, NodePort, LoadBalancer)
    
- Understanding Ingress
    
- Using and Configuring CoreDNS
    

🔧 **Lab:**

- Expose an application using different service types.
    
- Configure Ingress rules and DNS settings.
    

---

### **Part VI: Storage**

#### **Branch 7: Storage**

**Topics include:**

- Understanding Volumes and Persistent Volumes
    
- Static vs. Dynamic Provisioning
    
- Creating PersistentVolumes and PersistentVolumeClaims
    
- Understanding Storage Classes
    

🔧 **Lab:**

- Create and manage PersistentVolumes and PersistentVolumeClaims.
    
- Implement Storage Classes in a cluster.
    

---

### **Part VII: Troubleshooting**

#### **Branch 8: Troubleshooting Kubernetes Clusters**

**Topics include:**

- Evaluating Cluster and Node Logging
    
- Monitoring Cluster Components and Applications
    
- Troubleshooting Application Failures
    
- Troubleshooting Pods and Services
    
- Troubleshooting Cluster and Control Plane Failures
    

🔧 **Lab:**

- Simulate and troubleshoot common cluster issues.
    
- Practice using kubectl logs and kubectl exec for debugging.
    

---

### **Part VIII: Wrapping Up**

#### **Branch 9: Final Preparation and Mock Exams**

**Topics include:**

- Reviewing Exam Essentials
    
- Practicing with Mock Exams
    
- Using the Killer.sh Simulator
    

🔧 **Lab:**

- Take practice exams and identify areas for improvement.
    

---

## Tools and Lab Environment

To complete the labs, you will need the following tools:

- Minikube
    
- kubeadm
    
- kubectl
    
- Docker
    
- Helm
    
- Kustomize
    

---

## Getting Started

1. Clone the repository to your local machine.
    
2. Checkout the branch corresponding to the topic you want to study:
    

```
git checkout <branch-name>
```

3. Explore the Docs and Labs folders for study materials and exercises.
    
4. Follow the weekly study plan to systematically progress through the topics.
    

---

## Contributing

We welcome contributions to improve the guide. Follow these steps to contribute:

1. Fork the repository.
    
2. Create a new branch:
    

```
git checkout -b feature/YourFeature
```

3. Commit your changes:
    

```
git commit -m 'Add YourFeature'
```

4. Push your changes:
    

```
git push origin feature/YourFeature
```

5. Open a Pull Request.