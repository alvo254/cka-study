# Certified Kubernetes Administrator (CKA) Study Guide

A comprehensive repository of resources, configurations, and practical examples to help you prepare for and pass the Certified Kubernetes Administrator exam, structured according to GitOps principles.

## Table of Contents

- [Certified Kubernetes Administrator (CKA) Study Guide](#certified-kubernetes-administrator-cka-study-guide)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Repository Structure](#repository-structure)
  - [GitOps Approach](#gitops-approach)
  - [Quick Start Guide](#quick-start-guide)
  - [Documentation Resources](#documentation-resources)
  - [Kubernetes Configurations](#kubernetes-configurations)
  - [Practice Modules](#practice-modules)
    - [Module 1: RBAC \& Authentication](#module-1-rbac--authentication)
    - [Module 2: Workloads \& Scheduling](#module-2-workloads--scheduling)
    - [Module 3: Networking \& Services](#module-3-networking--services)
    - [Module 4: Storage \& Persistence](#module-4-storage--persistence)
  - [Kustomize Implementation](#kustomize-implementation)
    - [Kustomize Structure](#kustomize-structure)
    - [Applying Kustomizations](#applying-kustomizations)
    - [Environment-Specific Configurations](#environment-specific-configurations)
  - [Testing Environment Setup](#testing-environment-setup)
  - [Common Commands](#common-commands)
  - [Additional Resources](#additional-resources)
  - [Contributing](#contributing)
  - [License](#license)

## Introduction

This repository contains a structured collection of Kubernetes configurations, documentation, and hands-on exercises designed to help you master the concepts needed for the Certified Kubernetes Administrator (CKA) exam. The materials are organized into logical modules covering all major exam topics, with practical examples that can be applied in a real Kubernetes environment. The repository follows GitOps principles, storing all infrastructure configurations as code and using Kustomize for environment-specific adaptations.

## Repository Structure

```
.
├── backup-kind.yaml         # Backup configuration for Kind cluster
├── kind.yaml                # Kind cluster primary configuration
├── koornchart.yaml          # Helm chart configuration
├── Docs/                    # Documentation and reference materials
├── misc/                    # Supplementary Kubernetes manifests
├── pkg/                     # Practice modules with examples
│   ├── pkg1/                # RBAC & Authentication
│   ├── pkg2/                # Workloads & Scheduling
│   ├── pkg3/                # Networking & Services
│   └── pkg4/                # Storage & Persistence
├── kustomize/               # Kustomize configurations
│   ├── base/                # Base resources
│   └── overlays/            # Environment-specific overlays
│       ├── dev/             # Development environment
│       ├── staging/         # Staging environment
│       └── prod/            # Production environment
└── README.md                # This documentation
```

## GitOps Approach

This repository is structured following GitOps principles, which means:

1. **Declarative Infrastructure**: All Kubernetes resources are defined declaratively in YAML manifests.

2. **Version-Controlled Configuration**: All configurations are stored in Git, providing complete history, auditability, and the ability to revert changes if needed.

3. **Environment Segregation**: The repository uses Kustomize overlays to separate configurations for different environments while maintaining a single source of truth in the base configurations.

4. **Automated Synchronization**: While not implemented in this study repository, in a production setting, tools like ArgoCD or Flux could be configured to automatically apply changes from this repository to your clusters.

5. **Package Organization**: The repository is organized into logical modules (pkg1-4) to clearly separate different aspects of Kubernetes administration, making it easier to understand concepts incrementally.

This approach mirrors real-world Kubernetes administration practices, where configuration changes flow through a Git repository rather than being applied directly to clusters, enabling better governance, collaboration, and reliability.

## Quick Start Guide

```bash
# Clone this repository
git clone https://github.com/yourusername/cka-study.git
cd cka-study

# Create a local Kubernetes cluster using Kind
kind create cluster --config kind.yaml

# Apply the configurations from a specific module
kubectl apply -f pkg/pkg1/manifests/

# Apply a Kustomize configuration for development
kubectl apply -k kustomize/overlays/dev

# Monitor the resources
kubectl get all -A
```

## Documentation Resources

The `Docs/` directory contains comprehensive guides on key Kubernetes concepts:

| Document | Description |
|----------|-------------|
| `cheat-sheet.md` | Essential commands and quick reference for exam day |
| `comprehensive ingress controller.md` | Deep dive into Ingress controller setup and configuration |
| `comprehensive gateway API.md` | Complete reference for the Gateway API specification |
| `gatewayAPI.md` | Quick reference for Gateway API resources |
| `kubeadm-cluster-setup.md` | Step-by-step guide for setting up a production cluster |
| `port mappings.md` | Reference for common service port mappings |
| `setting up dnsmasq.md` | Guide for configuring DNS resolution |
| `volume guide.md` | Complete overview of Kubernetes storage options |

## Kubernetes Configurations

The repository includes production-ready Kubernetes configurations:

- **Kind Cluster**: Pre-configured Kind setup for local development (`kind.yaml`)
- **Helm Charts**: Example Helm chart with application deployment (`koornchart.yaml`)
- **Miscellaneous**: Additional configurations for networking, monitoring, and validation in the `misc/` directory:
  - `bgp-kind.yaml`: BGP network configuration
  - `checkoutservice.yaml`: Example microservice deployment
  - `datadog-agent.yaml`: Monitoring agent setup
  - `kubernetes-manifests.yaml`: Multi-resource manifest collection
  - `validate-resources.yaml`: Resource validation configuration

## Practice Modules

### Module 1: RBAC & Authentication

Practice implementing Kubernetes Role-Based Access Control, ServiceAccounts, and authentication mechanisms.

### Module 2: Workloads & Scheduling

Explore deployment strategies, scaling mechanisms, and resource management in Kubernetes.

### Module 3: Networking & Services

Master Kubernetes networking concepts, including Services, Ingress, and the new Gateway API.

### Module 4: Storage & Persistence

Learn how to implement persistent storage solutions in Kubernetes applications.

## Kustomize Implementation

This repository implements Kustomize to manage environment-specific configurations while maintaining a single source of truth in base resources. This approach is core to GitOps practices and is widely used in production Kubernetes environments.

### Kustomize Structure

The Kustomize configuration follows a base/overlay pattern:

```
kustomize/
├── base/                 # Common configurations across all environments
│   └── kustomization.yaml
└── overlays/             # Environment-specific customizations
    ├── dev/              # Development environment
    │   └── kustomization.yaml
    ├── staging/          # Staging environment
    │   └── kustomization.yaml
    └── prod/             # Production environment
        └── kustomization.yaml
```

The base kustomization includes core resources used across all environments, while each overlay specifies environment-specific customizations such as resource limits, replica counts, and additional components.

### Applying Kustomizations

To apply a specific environment configuration, use the `kubectl apply -k` command:

```bash
# Apply development environment configuration
kubectl apply -k kustomize/overlays/dev

# Apply staging environment configuration
kubectl apply -k kustomize/overlays/staging

# Apply production environment configuration
kubectl apply -k kustomize/overlays/prod
```

You can preview the resulting YAML before applying:

```bash
# Preview what will be applied to the cluster
kubectl kustomize kustomize/overlays/dev
```

### Environment-Specific Configurations

Each environment overlay includes specific adjustments appropriate for that environment:

- **Development**:
  - Lower resource limits
  - Single replicas
  - Development-specific ConfigMaps
  - Debug settings enabled

- **Staging**:
  - Medium resource allocations
  - Multiple replicas (2)
  - Ingress configurations for testing
  - Staging secrets and ConfigMaps

- **Production**:
  - Higher resource allocations
  - High availability configuration (3+ replicas)
  - Advanced networking (Gateway API)
  - Production secrets and ConfigMaps
  - Horizontal Pod Autoscaling

This approach enables you to practice how configurations evolve across environments while maintaining consistency in core resources.

## Testing Environment Setup

```bash
# Create a cluster with the provided configuration
kind create cluster --config kind.yaml

# Verify cluster is running
kubectl cluster-info

# Apply network policies (if needed)
kubectl apply -f misc/bgp-kind.yaml

# Set up monitoring (optional)
kubectl apply -f misc/datadog-agent.yaml

# Apply development environment configuration
kubectl apply -k kustomize/overlays/dev
```

## Common Commands

```bash
# View all resources in the cluster
kubectl get all --all-namespaces

# Apply all manifests in a directory
kubectl apply -f pkg/pkg1/manifests/

# Apply a specific kustomization
kubectl apply -k kustomize/overlays/dev

# Preview a kustomization output
kubectl kustomize kustomize/overlays/staging

# Watch pods as they deploy
kubectl get pods -w

# Check logs of a specific pod
kubectl logs pod-name -n namespace-name

# Execute commands inside a container
kubectl exec -it pod-name -- /bin/bash
```

## Additional Resources

- [Official Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [CKA Exam Curriculum](https://github.com/cncf/curriculum)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [GitOps Principles](https://www.gitops.tech/)

## Contributing

Contributions to improve the examples, documentation, or add new practice exercises are welcome. Please submit a pull request or open an issue to discuss your ideas.

## License

This project is licensed under the MIT License - see the LICENSE file for details.


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


