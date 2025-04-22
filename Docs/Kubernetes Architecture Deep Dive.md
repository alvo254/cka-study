
# Kubernetes Architecture Deep Dive: From API to Kubelet and CRI

## 1. Introduction to Kubernetes Architecture

Kubernetes (k8s) is a portable, extensible, open-source platform for managing containerized workloads and services that facilitates both declarative configuration and automation. The name Kubernetes originates from Greek, meaning helmsman or pilot, with k8s being an abbreviation derived from counting the eight letters between the "k" and the "s".

At its core, Kubernetes provides a framework to run distributed systems resiliently. It takes care of scaling and failover for applications, provides deployment patterns, and helps manage the lifecycle of containerized applications.

## 2. High-Level Architecture Overview

Kubernetes follows a client-server architecture with multiple components that work together to maintain the desired state of the cluster. The architecture can be divided into two main parts:

### 2.1 Control Plane (Master Components)

The Control Plane is the brain of the Kubernetes cluster, responsible for making global decisions about the cluster (such as scheduling), detecting and responding to cluster events.

Key components include:

#### 2.1.1 kube-apiserver

The API server is the front-end of the Kubernetes control plane, exposing the Kubernetes API. It's designed to scale horizontally by deploying more instances and load balancing between them.

The API server processes RESTful API requests, validates them, and updates the corresponding objects in etcd. It serves as the gateway for all external and internal clients, including:

- Command-line tools (kubectl)
- Web UI (Kubernetes Dashboard)
- Other control plane components

The API server implements a watch mechanism allowing clients to subscribe to changes to resources they're interested in, enabling event-driven architectures.

#### 2.1.2 etcd

etcd is a consistent and highly-available key-value store used as Kubernetes' backing store for all cluster data. It stores the configuration data, state, and metadata for the entire cluster.

Key characteristics:

- Distributed: Replicates data across multiple nodes
- Strongly consistent: Uses the Raft consensus algorithm
- Provides a watch feature that notifies clients of changes
- Supports versioning of keys
- Implements leases for TTL-based keys

Kubernetes uses etcd to store:

- Node information (compute resources, health, etc.)
- Pod definitions and statuses
- Namespaces, services, and endpoints
- ConfigMaps and secrets
- StatefulSet and Deployment configurations
- Role-based access control (RBAC) data

#### 2.1.3 kube-scheduler

The scheduler watches for newly created Pods with no assigned node and selects a node for them to run on based on various factors:

- Resource requirements
- Hardware/software/policy constraints
- Affinity and anti-affinity specifications
- Data locality
- Inter-workload interference
- Deadlines

The scheduling process consists of two phases:

1. **Filtering**: Finds the set of nodes where it's feasible to schedule the Pod
2. **Scoring**: Ranks the remaining nodes to find the best placement

The scheduler is pluggable, allowing custom algorithms through the scheduler framework.

#### 2.1.4 kube-controller-manager

The controller manager runs controller processes that regulate the state of the system. Logically, each controller is a separate process, but they're compiled into a single binary and run in a single process to reduce complexity.

Key controllers include:

- **Node Controller**: Notices and responds when nodes go down
- **Replication Controller**: Maintains the correct number of Pods for every ReplicationController object
- **Endpoints Controller**: Populates the Endpoints object (joins Services & Pods)
- **Service Account & Token Controllers**: Create default accounts and API access tokens for new namespaces
- **Deployment Controller**: Manages Deployment objects and creates ReplicaSets
- **StatefulSet Controller**: Manages StatefulSet objects and ensures ordered deployment
- **Job Controller**: Watches for Job objects and creates Pods to run them
- **CronJob Controller**: Creates Job objects according to schedule

#### 2.1.5 cloud-controller-manager

When running in a cloud environment, this component embeds cloud-specific control logic, allowing the core Kubernetes code to remain cloud-agnostic. It runs controllers that interact with the underlying cloud providers:

- **Node Controller**: Checks the cloud provider to determine if a node has been deleted
- **Route Controller**: Sets up routes in the cloud infrastructure
- **Service Controller**: Creates, updates, and deletes cloud provider load balancers
- **Volume Controller**: Creates, attaches, and mounts volumes, and interacts with the cloud provider to orchestrate volumes

### 2.2 Node Components

Node components run on every worker node, maintaining running pods and providing the Kubernetes runtime environment.

#### 2.2.1 kubelet

The kubelet is the primary "node agent" that runs on each node. It ensures containers are running in a Pod by:

- Taking a set of PodSpecs provided through various mechanisms
- Ensuring the containers described in those PodSpecs are running and healthy
- Reporting the status of the node and each Pod to the API server

The kubelet doesn't manage containers which were not created by Kubernetes.

#### 2.2.2 kube-proxy

kube-proxy is a network proxy that runs on each node, implementing part of the Kubernetes Service concept. It maintains network rules on nodes that allow network communication to Pods from inside or outside of the cluster.

kube-proxy uses the operating system packet filtering layer if available, otherwise, it forwards the traffic itself. It can operate in different modes:

- **IPVS mode**: Uses the Linux kernel IPVS (IP Virtual Server) for load balancing
- **iptables mode**: Uses Linux iptables rules for routing
- **userspace mode**: (legacy) Uses a userspace proxy implementation

#### 2.2.3 Container Runtime

The container runtime is the software responsible for running containers. Kubernetes supports various runtimes through the Container Runtime Interface (CRI):

- containerd
- CRI-O
- Docker (via cri-dockerd, as dockershim was removed in Kubernetes 1.24)
- Any implementation of the CRI

## 3. Kubernetes API Resources and Types

Kubernetes defines a rich set of API resources that represent primitive objects within the system. These resources are organized by API groups and versions.

### 3.1 Core API Resources

The core API resources include:

#### 3.1.1 Pods (core/v1)

The smallest deployable units that can be created and managed in Kubernetes. A Pod represents a set of running containers on your cluster, typically encapsulating a single application instance.

Key aspects:

- Pods share a network namespace and can communicate via localhost
- Pods can share storage volumes
- Pods are ephemeral by design
- Pods have a lifecycle (Pending, Running, Succeeded, Failed, Unknown)
- Pods have a unique IP address within the cluster network

#### 3.1.2 Services (core/v1)

An abstract way to expose an application running on a set of Pods as a network service. Services enable loose coupling between dependent Pods.

Types:

- **ClusterIP**: Exposes the Service on a cluster-internal IP (default)
- **NodePort**: Exposes the Service on each Node's IP at a static port
- **LoadBalancer**: Exposes the Service externally using a cloud provider's load balancer
- **ExternalName**: Maps the Service to a DNS name
- **Headless**: Used when direct endpoint connections are desired

#### 3.1.3 Persistent Volumes (core/v1)

A piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes.

Components:

- **PersistentVolume (PV)**: A piece of storage in the cluster
- **PersistentVolumeClaim (PVC)**: A request for storage by a user
- **StorageClass**: Provides a way for administrators to describe the "classes" of storage they offer

### 3.2 Workload Resources

Resources that manage the lifecycle of Pods:

#### 3.2.1 ReplicaSet (apps/v1)

Ensures that a specified number of Pod replicas are running at any given time. It's often used by Deployments as a mechanism to replace Pods.

#### 3.2.2 Deployment (apps/v1)

Provides declarative updates for Pods and ReplicaSets. It allows for:

- Rolling updates and rollbacks
- Scale up/down capability
- Pause and resume of deployments
- Clean up of older ReplicaSets

#### 3.2.3 StatefulSet (apps/v1)

Manages the deployment and scaling of a set of Pods, providing guarantees about the ordering and uniqueness of these Pods. Suitable for applications that require:

- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

#### 3.2.4 DaemonSet (apps/v1)

Ensures that all (or some) nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed, those Pods are garbage collected.

Uses:

- Running a cluster storage daemon
- Running a logs collection daemon on every node
- Running a node monitoring daemon on every node

#### 3.2.5 Job and CronJob (batch/v1)

A Job creates one or more Pods and ensures that a specified number of them successfully terminate.

A CronJob creates Jobs on a repeating schedule.

### 3.3 Configuration Resources

#### 3.3.1 ConfigMap (core/v1)

Used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as:

- Environment variables
- Command-line arguments
- Configuration files in a volume

#### 3.3.2 Secret (core/v1)

Used to store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys.

Types:

- **Opaque**: Arbitrary user-defined data
- **kubernetes.io/service-account-token**: Service account token
- **kubernetes.io/dockerconfigjson**: Serialized ~/.docker/config.json file
- **kubernetes.io/tls**: Data for a TLS client or server
- **bootstrap.kubernetes.io/token**: Bootstrap token data

### 3.4 Networking Resources

#### 3.4.1 Ingress (networking.k8s.io/v1)

An API object that manages external access to the services in a cluster, typically HTTP/HTTPS. Ingress provides:

- Load balancing
- SSL termination
- Name-based virtual hosting

#### 3.4.2 NetworkPolicy (networking.k8s.io/v1)

A specification of how groups of Pods are allowed to communicate with each other and other network endpoints.

## 4. API Request Flow

The journey of a Kubernetes API request involves multiple components working together. Here's a detailed look at the flow:

### 4.1 Request Initiation

1. A client (kubectl, programmatic client, or another component) formulates an API request
2. The request includes:
    - HTTP method (GET, POST, PUT, DELETE, PATCH)
    - Target resource URL (e.g., /api/v1/namespaces/default/pods)
    - Request body (for creation/update operations)
    - Authentication credentials

### 4.2 Authentication

The kube-apiserver processes the request through several authentication plugins:

1. Client certificate authentication
2. Bearer token authentication
3. Basic authentication
4. Service account tokens
5. OpenID Connect tokens
6. Webhook token authentication

If authentication fails, the server returns a 401 Unauthorized response.

### 4.3 Authorization

After authentication, the request undergoes authorization through the configured authorization mode(s):

1. **RBAC (Role-Based Access Control)**: Checks if the user has permission to perform the action on the resource
2. **ABAC (Attribute-Based Access Control)**: Uses policies based on attributes
3. **Node**: Special-purpose authorizer that grants permissions to kubelets
4. **Webhook**: Offloads authorization decisions to an external HTTP(S) service

If authorization fails, the server returns a 403 Forbidden response.

### 4.4 Admission Control

If the request is authenticated and authorized, it passes through admission controllers:

1. **MutatingAdmissionWebhook**: Can modify the request object
2. **ValidatingAdmissionWebhook**: Can validate but not modify the request
3. Built-in admission controllers like:
    - **LimitRanger**: Sets resource limits
    - **ResourceQuota**: Enforces namespace resource quotas
    - **ServiceAccount**: Automates service account management
    - **DefaultStorageClass**: Sets the default storage class
    - **PodSecurity**: Enforces Pod Security Standards

### 4.5 Validation and Persistence

If the request passes admission control:

1. The API server validates the request object against the schema
2. For write operations (POST, PUT, PATCH), the API server persists the object to etcd
3. etcd stores the object under a key matching its resource type and name
4. etcd notifies the API server that the write completed successfully

### 4.6 Response and Watch Notifications

1. The API server formulates a response to the client
2. For watch requests, the API server establishes a long-lived connection
3. The API server notifies controllers and other components watching the resource

## 5. Scheduling Flow

When a new Pod is created, the scheduling process follows these steps:

### 5.1 Pod Creation and Admission

1. A client creates a Pod by sending a request to the API server
2. The Pod enters the Pending state in the API server
3. The scheduler watches for Pods with no node assignment

### 5.2 Scheduler Processing

1. The scheduler retrieves the Pod specification from the API server
2. The scheduler runs the Pod through a series of predicates (filtering):
    - **PodFitsResources**: Checks if the node has enough resources
    - **PodFitsHostPorts**: Ensures no port conflicts
    - **PodMatchNodeSelector**: Honors node selectors
    - **NoVolumeZoneConflict**: Checks volume zone restrictions
    - **CheckNodeDiskPressure**: Avoids nodes with disk pressure
    - Many others
3. The scheduler then runs the Pod through priority functions (scoring):
    - **LeastRequestedPriority**: Favors nodes with fewer resource requests
    - **BalancedResourceAllocation**: Balances CPU and memory utilization
    - **ImageLocalityPriority**: Prefers nodes that already have required images
    - **InterPodAffinityPriority**: Implements pod affinity/anti-affinity
    - Many others
4. The scheduler combines the scores and selects the highest-scoring node
5. The scheduler updates the Pod definition with the selected node name

### 5.3 Node Assignment

1. The API server persists the updated Pod definition with the node assignment to etcd
2. The kubelet on the selected node is notified of the new Pod assignment via its watch on the API server

## 6. Kubelet Deep Dive

The kubelet is one of the most complex components in Kubernetes, responsible for managing containers on a node.

### 6.1 Kubelet Architecture

The kubelet's architecture consists of several core components:

#### 6.1.1 Pod Lifecycle Event Generator (PLEG)

The PLEG component:

- Monitors container state changes
- Generates events for the kubelet to process
- Reconciles the desired state with the actual state

#### 6.1.2 SyncLoop

The main control loop of the kubelet that:

- Watches for Pod spec changes
- Ensures Pods and their containers are running
- Reports status back to the API server

#### 6.1.3 Volume Manager

Manages the mounting and unmounting of volumes:

- Attaches volumes to the node
- Mounts volumes into Pod containers
- Handles cleanup of volumes

#### 6.1.4 Image Manager

Responsible for:

- Pulling images from registries
- Managing the local image cache
- Garbage collecting unused images

#### 6.1.5 Eviction Manager

Monitors resource usage and evicts Pods when the node is under resource pressure:

- Memory pressure
- Disk pressure
- PID pressure

#### 6.1.6 CAdvisor (Container Advisor)

A built-in agent that collects, aggregates, processes, and exports information about running containers:

- CPU usage
- Memory usage
- Network statistics
- Filesystem usage

### 6.2 Kubelet Configuration

The kubelet is configured through:

- Command-line flags
- Configuration file (KubeletConfiguration)
- Dynamic configuration via the API server

Key configuration parameters include:

- `--config`: Path to a configuration file
- `--container-runtime-endpoint`: Endpoint of CRI container runtime service
- `--kubeconfig`: Path to credential file for API server
- `--node-status-update-frequency`: Frequency of node status updates
- `--max-pods`: Maximum number of Pods that can run on this node
- `--eviction-hard`: Signals that trigger hard evictions

### 6.3 Pod Lifecycle Management

The kubelet manages Pods through their entire lifecycle:

#### 6.3.1 Pod Admission

When a Pod is assigned to a node, the kubelet:

1. Retrieves the Pod specification from the API server
2. Passes the Pod through admission handlers:
    - **Capabilities**: Validates security capabilities
    - **AppArmor**: Applies AppArmor profiles
    - **NoNewPrivs**: Prevents privilege escalation
    - **EventedPLEG**: Updates the Pod Lifecycle Event Generator

#### 6.3.2 Pod Creation

The kubelet creates the Pod by:

1. Creating the Pod directory structure
2. Pulling required images
3. Setting up volumes
4. Creating the Pod sandbox via CRI
5. Creating and starting containers via CRI

#### 6.3.3 Pod Monitoring

The kubelet continuously monitors Pods:

1. Polls container runtimes for container statuses
2. Runs liveness, readiness, and startup probes
3. Reconciles desired state with actual state
4. Reports Pod status to the API server

#### 6.3.4 Pod Termination

When a Pod is terminated, the kubelet:

1. Terminates containers gracefully (respecting terminationGracePeriodSeconds)
2. Runs container preStop hooks if defined
3. Unmounts volumes
4. Removes the Pod sandbox
5. Cleans up Pod resources

### 6.4 Kubelet Health Management

The kubelet monitors its own health and reports to the API server:

- Node condition (Ready, DiskPressure, MemoryPressure, PIDPressure, NetworkUnavailable)
- Node status (capacity, allocatable resources, addresses, conditions)
- Node heartbeats (updates the node's .status.conditions)

## 7. Container Runtime Interface (CRI)

The Container Runtime Interface (CRI) is a plugin interface which enables the kubelet to use different container runtimes without recompilation.

### 7.1 CRI Architecture

The CRI architecture consists of:

#### 7.1.1 Protocol Definition

- Uses Protocol Buffers (protobuf) for service definition
- Defines the gRPC service interface in `api.proto`
- Located in the Kubernetes source at `pkg/kubelet/apis/cri/runtime/v1/api.proto`

#### 7.1.2 Services

The CRI defines two services:

1. **RuntimeService**: For container and Pod lifecycle management
    
    - Pod operations (create, delete, list, status)
    - Container operations (create, start, stop, remove, list, status)
    - Exec, port forwarding, and container stats
2. **ImageService**: For image operations
    
    - List images
    - Pull images
    - Remove images
    - Image status

#### 7.1.3 CRI Implementations

Major CRI implementations include:

1. **containerd**:
    
    - Default runtime in many Kubernetes distributions
    - Uses the `cri` plugin (built-in since containerd 1.1)
    - Communicates with containerd via its internal APIs
    - Uses runc as the default low-level runtime
2. **CRI-O**:
    
    - Lightweight implementation designed specifically for Kubernetes
    - Optimized for Kubernetes
    - Also uses OCI-compliant runtimes like runc
    - Direct integration with OCI image registries
3. **Docker** (via cri-dockerd):
    
    - Legacy approach, as dockershim was removed in Kubernetes 1.24
    - cri-dockerd is an adapter that implements CRI for Docker

### 7.2 CRI Protocol Details

#### 7.2.1 Key Data Structures

**PodSandboxConfig**:

- Contains Pod-level configuration
- Includes metadata, labels, annotations
- Network configuration
- Linux-specific options (namespaces, cgroups, security context)

**ContainerConfig**:

- Contains container-level configuration
- Includes metadata, image, command
- Mounts, devices, and environment variables
- Resource requirements
- Security settings

#### 7.2.2 Primary Methods

**Runtime Service**:

- `RunPodSandbox`: Creates and starts a Pod sandbox
- `StopPodSandbox`: Stops a Pod sandbox
- `RemovePodSandbox`: Removes a Pod sandbox
- `CreateContainer`: Creates a container within a Pod sandbox
- `StartContainer`: Starts a created container
- `StopContainer`: Stops a running container
- `RemoveContainer`: Removes a container
- `ListContainers`: Lists containers by filter
- `ContainerStatus`: Returns status of a container
- `UpdateContainerResources`: Updates container resources
- `ExecSync`: Runs a command in a container synchronously
- `Exec`: Runs a command in a container asynchronously
- `Attach`: Attaches to a running container
- `PortForward`: Forward ports from a Pod sandbox

**Image Service**:

- `ListImages`: Lists images by filter
- `ImageStatus`: Returns status of an image
- `PullImage`: Pulls an image from a registry
- `RemoveImage`: Removes an image
- `ImageFsInfo`: Returns storage information for images

### 7.3 CRI Workflow

Let's examine the detailed workflow when a Pod is created:

#### 7.3.1 Pod Sandbox Creation

When creating a Pod, the kubelet:

1. Constructs a `PodSandboxConfig` including:
    
    - Pod metadata (name, namespace, UID)
    - Labels and annotations
    - DNS configuration
    - Port mappings
    - Security context
    - Linux namespace options
2. Calls `RunPodSandbox` with this configuration
    
3. The CRI implementation creates the Pod sandbox:
    
    - Sets up namespaces (network, IPC, UTS)
    - Configures cgroups for resource isolation
    - Configures networking (typically by calling CNI plugins)
    - Creates a "pause" container (in some implementations)
4. The CRI implementation returns the Pod sandbox ID
    

#### 7.3.2 Container Creation

For each container in the Pod spec, the kubelet:

1. Constructs a `ContainerConfig` including:
    
    - Container metadata (name, attempt)
    - Image information
    - Command and arguments
    - Environment variables
    - Mounts (volumes, secrets, configmaps)
    - Devices
    - Resource limits
    - Security context
    - Linux-specific options
2. Calls `CreateContainer` with this configuration and the Pod sandbox ID
    
3. The CRI implementation:
    
    - Pulls the container image if needed
    - Prepares the root filesystem
    - Sets up mount points
    - Creates the container but doesn't start it
4. The CRI implementation returns the container ID
    

#### 7.3.3 Container Start

After creating each container, the kubelet:

1. Calls `StartContainer` with the container ID
    
2. The CRI implementation:
    
    - Starts the container
    - Sets up process-level security
    - Applies resource constraints
    - Executes the container command

#### 7.3.4 Ongoing Monitoring

The kubelet continuously monitors containers:

1. Periodically calls `ListContainers` and `ListPodSandbox` to get overall state
2. Calls `ContainerStatus` for detailed information about specific containers
3. Calls `PodSandboxStatus` for Pod-level status
4. Uses this information to:
    - Update the Pod status in the API server
    - Enforce restart policies
    - Execute health checks (liveness, readiness, startup probes)

### 7.4 CRI Security Aspects

The CRI implementation handles several security aspects:

#### 7.4.1 Container Security Context

Passed through the `SecurityContext` field in `ContainerConfig`:

- Run as user/group
- Privileged mode
- Capabilities add/drop
- SELinux options
- AppArmor profile
- Seccomp profile
- No new privileges flag
- ReadOnly root filesystem

#### 7.4.2 Pod Security Context

Applied at the Pod level:

- Host network, PID, IPC sharing
- Sysctls
- FS group
- Shared process namespace

### 7.5 CRI Storage Integration

Storage integration occurs through:

#### 7.5.1 Volumes

The kubelet prepares volumes and passes mount information to the CRI:

- Host path mounts
- EmptyDir volumes
- Persistent volumes already attached and mounted by the kubelet
- ConfigMap and Secret volumes prepared by the kubelet

#### 7.5.2 CSI Integration

The Container Storage Interface (CSI) works alongside the CRI:

1. Kubernetes CSI sidecars attach and mount volumes
2. The kubelet passes the mounted volumes to the CRI
3. The CRI propagates these mounts into containers

## 8. Networking Stack

Kubernetes networking involves multiple layers working together:

### 8.1 Pod Networking

Every Pod gets its own IP address, allowing Pods to communicate with each other without NAT:

#### 8.1.1 Container Network Interface (CNI)

1. The kubelet creates the Pod sandbox via CRI
2. The CRI runtime creates network namespaces
3. The kubelet calls CNI plugins to set up networking
4. CNI plugins:
    - Allocate IP addresses
    - Set up routes
    - Configure interfaces
    - Program iptables or other packet processing systems

#### 8.1.2 CNI Plugins

Common CNI plugins include:

- **Calico**: Uses BGP for routing and supports network policies
- **Cilium**: Uses eBPF for networking and security
- **Flannel**: Simple overlay network
- **Weave Net**: Creates a virtual network across nodes

### 8.2 Service Networking

Services abstract Pod access through:

#### 8.2.1 Service Implementation

1. The API server creates a Service object
2. The endpoint controller watches for Service and Pod changes
3. The endpoint controller creates Endpoints objects
4. kube-proxy watches for Service and Endpoints changes
5. kube-proxy programs the host networking (iptables, IPVS, etc.)
6. Traffic to the Service IP is redirected to backend Pods

#### 8.2.2 DNS Integration

CoreDNS (or kube-dns in older versions):

1. Watches for Service and Endpoint changes
2. Creates DNS records for Services
3. Resolves Service names to cluster IPs
4. Enables service discovery within the cluster

## 9. Complete Pod Lifecycle: A Full Example

Let's walk through the complete lifecycle of a Pod from creation to termination:

### 9.1 Pod Creation Request

1. User submits a Pod YAML via kubectl:
    
    ```
    kubectl apply -f pod.yaml
    ```
    
2. kubectl:
    
    - Validates the YAML format
    - Authenticates to the API server
    - Sends a POST request to `/api/v1/namespaces/default/pods`
3. API server:
    
    - Authenticates the request
    - Authorizes the request against RBAC policies
    - Runs admission controllers
    - Validates the Pod spec
    - Assigns a UID to the Pod
    - Sets initial status to "Pending"
    - Persists the Pod object to etcd
    - Returns success to the client

### 9.2 Scheduling

1. The scheduler:
    
    - Watches for unscheduled Pods
    - Retrieves the new Pod
    - Runs filtering algorithms to find eligible nodes
    - Runs scoring algorithms to rank eligible nodes
    - Selects the highest-scoring node (Node1)
    - Updates the Pod with `nodeName: Node1`
    - Persists the change via the API server
2. API server:
    
    - Validates the update
    - Stores the updated Pod in etcd

### 9.3 Kubelet Processing

1. The kubelet on Node1:
    
    - Watches for Pods assigned to its node
    - Sees the new Pod assignment
    - Retrieves the complete Pod spec
    - Admits the Pod through admission handlers
    - Creates a directory structure for the Pod
    - Integrates with the volume manager to set up volumes
2. Volume preparation:
    
    - The kubelet's volume manager handles volume operations
    - PersistentVolumes are attached to the node
    - Volumes are mounted to the appropriate paths
    - Secret and ConfigMap volumes are populated

### 9.4 CRI Interactions

1. Pod sandbox creation:
    
    - Kubelet constructs a `PodSandboxConfig`
    - Kubelet calls RuntimeService `RunPodSandbox()`
    - CRI runtime creates the Pod sandbox with namespaces
    - CRI runtime creates the "pause" container
    - CNI plugins are invoked to set up networking
    - Pod sandbox ID is returned to the kubelet
2. For each init container:
    
    - Kubelet checks if image is present via ImageService `ImageStatus()`
    - If not present, kubelet calls ImageService `PullImage()`
    - Kubelet calls RuntimeService `CreateContainer()`
    - Kubelet calls RuntimeService `StartContainer()`
    - Kubelet waits for completion before proceeding to the next container
    - Status is reported back to the API server
3. For each regular container:
    
    - Kubelet follows the same process for image pulling
    - Kubelet calls RuntimeService `CreateContainer()`
    - Container config includes:
        - Commands and arguments
        - Environment variables
        - Volume mounts
        - Resource limits
        - Probe definitions
    - Kubelet calls RuntimeService `StartContainer()`
    - Container status is monitored and reported to the API server

### 9.5 Container Runtime Operations

The CRI implementation (containerd, CRI-O) performs:

1. Image management:
    
    - Checking image presence in local storage
    - Connecting to registries
    - Authenticating if needed
    - Pulling and unpacking image layers
    - Managing the local image cache
2. Container creation:
    
    - Creating OCI runtime bundles
    - Setting up the root filesystem
    - Preparing mount points
    - Configuring namespaces
3. Container execution:
    
    - Invoking the low-level runtime (runc, runsc, etc.)
    - Setting up cgroups for resource limits
    - Configuring security profiles
    - Starting the container processes
    - Connecting stdio streams
4. Container monitoring:
    
    - Watching container state
    - Managing container lifecycle (restart policies)
    - Collecting container metrics

### 9.6 Pod Running State

1. Status reporting:
    
    - Kubelet periodically calls CRI methods to get container status
    - Kubelet updates Pod status in the API server
    - Phase changes from "Pending" to "Running"
    - Container statuses include start time, restart count
2. Health monitoring:
    
    - Kubelet executes liveness probes via CRI `ExecSync()`
    - Kubelet executes readiness probes
    - Kubelet executes startup probes
    - Results affect Pod conditions and potential restarts
3. Resource monitoring:
    
    - CAdvisor collects container metrics
    - Kubelet exposes these metrics via its `/metrics` endpoint
    - Metrics are scraped by monitoring solutions

### 9.7 Pod Termination

When a Pod is deleted (via kubectl delete, controller action, or node drain):

1. API server:
    
    - Adds deletion timestamp to the Pod
    - Pod enters "Terminating" state
    - Deletion grace period begins (default 30s)
2. Kubelet:
    
    - Notices the deletion timestamp
    - Executes preStop hooks via CRI
    - Gives containers time to shut down gracefully
3. For each container:
    
    - Kubelet calls RuntimeService `StopContainer()` with timeout
    - CRI runtime sends SIGTERM to the container
    - If container doesn't terminate, SIGKILL is sent after grace period
4. Cleanup:
    
    - Kubelet calls RuntimeService `RemoveContainer()` for each container
    - Kubelet calls RuntimeService `StopPodSandbox()`
    - Kubelet calls RuntimeService `RemovePodSandbox()`
    - Kubelet's volume manager unmounts and detaches volumes
    - Kubelet removes Pod directory structure
    - Kubelet reports final status to API server
5. API server:
    
    - Finalizes deletion from etcd
    - Pod no longer exists in the system

### 10.1 Custom CRI Features (Continued)

#### 10.1.2 Runtime Classes (Continued)

4. The CRI runtime selects the appropriate runtime configuration based on the handler
5. Enables different container runtimes or configurations per Pod
    - Example: Using gVisor for security-sensitive Pods
    - Example: Using kata-containers for hardware-level isolation

#### 10.1.3 Container Metrics

The CRI provides container metrics through:

- `ContainerStats`: Returns single container statistics
- `ListContainerStats`: Returns statistics for multiple containers
- Metrics include CPU, memory, filesystem, and network usage
- These metrics flow to the Metrics API and can be used by the Horizontal Pod Autoscaler

### 10.2 Kubelet and CRI Security

#### 10.2.1 Security Context Implementation

When a Pod or container defines security context options:

1. The kubelet translates them into CRI security configuration
2. The CRI implementation translates them into runtime-specific settings
3. The container runtime applies:
    - Linux capabilities
    - SELinux labels
    - AppArmor profiles
    - Seccomp filters
    - User namespace mappings
    - Privileged container settings

Example flow for a Pod with a seccomp profile:

1. Pod spec includes `securityContext.seccompProfile.type: "Localhost"`
2. Kubelet resolves the profile path on the node
3. Kubelet includes this in the CRI `SecurityContext`
4. CRI runtime sets up the seccomp profile for the container
5. OCI runtime applies seccomp filters to container processes

#### 10.2.2 Node Authorization

For node security:

1. Kubelets authenticate to the API server using TLS client certificates
2. The Node Authorizer (using the NodeRestriction admission plugin) limits kubelet permissions
3. Kubelets can only modify:
    - Pods bound to their node
    - Node status for their own node
    - Pod status updates for Pods on their node

### 10.3 Container Runtime Implementations in Detail

#### 10.3.1 containerd

Architecture details:

1. **containerd daemon**: Core container runtime
2. **CRI plugin**: Implements CRI for Kubernetes
3. **containerd-shim**: Process that owns container processes
4. **runc**: Low-level OCI-compliant runtime

When kubelet calls CRI methods:

1. CRI plugin receives the requests
2. Requests are translated to containerd API calls
3. containerd processes the requests
4. For container execution:
    - containerd creates OCI bundle
    - containerd spawns containerd-shim
    - containerd-shim uses runc to create container
    - containerd-shim monitors container processes
    - Container stdout/stderr are redirected to the shim

Advantages:

- Lower resource usage compared to Docker
- Reduced attack surface
- Direct implementation of CRI without adaptation layers
- Better isolation between daemon and containers

#### 10.3.2 CRI-O

Architecture details:

1. **CRI-O daemon**: Implements the CRI
2. **conmon**: Container monitor process
3. **OCI runtime**: Low-level runtime (runc by default)
4. **Storage layer**: Based on containers/storage
5. **Image layer**: Based on containers/image

When kubelet calls CRI methods:

1. CRI-O daemon receives the requests
2. For container execution:
    - CRI-O pulls images if needed
    - CRI-O creates OCI bundle
    - CRI-O spawns conmon process
    - conmon uses OCI runtime to create container
    - conmon monitors container processes
    - Container stdout/stderr are redirected to conmon

Advantages:

- Designed specifically for Kubernetes
- Minimal feature set focused on Kubernetes needs
- Direct integration with OCI standards
- No dependency on larger container ecosystem

### 10.4 Advanced Networking Topics

#### 10.4.1 CNI Interface Details

When setting up Pod networking:

1. Kubelet creates the Pod sandbox via CRI
    
2. Kubelet calls CNI plugins with:
    
    - Command (ADD, DEL, CHECK)
    - Container ID
    - Network namespace path
    - Network configuration
    - Extra arguments
3. CNI plugin responsibilities:
    
    - Interface creation and configuration
    - IP address allocation and assignment
    - Route setup
    - DNS configuration
    - Return IP addressing information to kubelet
4. CNI configuration:
    
    - Stored in `/etc/cni/net.d/`
    - Defines plugin chains and configuration
    - Can include multiple plugins (main, IPAM, meta-plugins)

#### 10.4.2 DNS Resolution Flow

For in-cluster DNS resolution:

1. Container's `/etc/resolv.conf` is configured to use the cluster DNS service
2. DNS queries flow to kube-dns or CoreDNS
3. CoreDNS processes the query:
    - Service names: `<service>.<namespace>.svc.cluster.local`
    - Pod IPs: `<ip-with-dashes>.<namespace>.pod.cluster.local`
4. CoreDNS plugins handle different resolution types:
    - **kubernetes**: Resolves Kubernetes service and Pod DNS
    - **forward**: Forwards external queries to upstream DNS
    - **cache**: Caches responses
    - **errors**: Handles error cases
    - **health**: Exposes health check endpoint

#### 10.4.3 Service Implementation Deep Dive

When a Service is created:

1. API server stores the Service object in etcd
    
2. Endpoint controller creates Endpoints object
    
3. kube-proxy implementation varies by mode:
    
    - **iptables mode**:
        - Creates KUBE-SERVICES chain
        - Adds rules mapping Service IP to KUBE-SVC-XXX chains
        - Creates KUBE-SVC-XXX chains with probability rules for load balancing
        - Each backend has a KUBE-SEP-XXX chain that DNATs to Pod IP
    - **IPVS mode**:
        - Creates IPVS virtual servers for each Service IP
        - Adds Pod IPs as real servers
        - Uses IPVS load balancing algorithms (rr, lc, dh, sh, sed, nq)
        - More efficient for large clusters than iptables
4. For external access:
    
    - **NodePort**: kube-proxy opens ports on all nodes
    - **LoadBalancer**: cloud-controller-manager provisions external load balancer
    - **Ingress**: Ingress controller sets up reverse proxy

### 10.5 Kubelet Garbage Collection

The kubelet performs garbage collection to free up resources:

#### 10.5.1 Container Garbage Collection

1. Configured via:
    
    - `--minimum-container-ttl-duration`
    - `--maximum-dead-containers-per-container`
    - `--maximum-dead-containers`
2. Container GC process:
    
    - Removes terminated containers
    - Preserves the most recent terminated containers
    - Prioritizes removing older containers
3. Implementation:
    
    - Kubelet calls CRI `ListContainers()` with filter for terminated containers
    - Identifies containers that exceed TTL
    - Calls `RemoveContainer()` for eligible containers

#### 10.5.2 Image Garbage Collection

1. Configured via:
    
    - `--image-gc-high-threshold`
    - `--image-gc-low-threshold`
2. Image GC process:
    
    - Triggered when disk usage exceeds high threshold
    - Removes unused images until usage falls below low threshold
    - Prioritizes removing older, unused images
3. Implementation:
    
    - Kubelet calls CRI `ImageFsInfo()` to check disk usage
    - Calls `ListImages()` to get all images
    - Determines unused images (not referenced by containers)
    - Calls `RemoveImage()` for eligible images

### 10.6 CRI Streaming Operations

The CRI supports streaming operations for interactive container access:

#### 10.6.1 Exec Implementation

When a user runs `kubectl exec`:

1. kubectl sends request to API server's exec endpoint
2. API server validates and authorizes the request
3. API server upgrades connection to SPDY
4. Request is proxied to the kubelet
5. Kubelet calls CRI `Exec()` method
6. CRI runtime starts exec process in the container
7. Bidirectional streams connect kubelet, CRI runtime, and container
8. SPDY streams connect kubelet and API server
9. kubectl connects to these streams

Implementation details:

- CRI `Exec()` returns a URL for the exec streaming server
- Kubelet connects to this URL to establish streams
- Streams are multiplexed over a single connection
- Four streams: stdin, stdout, stderr, and control

#### 10.6.2 Port Forward Implementation

When a user runs `kubectl port-forward`:

1. kubectl sends request to API server's port-forward endpoint
2. API server validates and authorizes the request
3. API server upgrades connection to SPDY
4. Request is proxied to the kubelet
5. Kubelet calls CRI `PortForward()` method
6. CRI runtime sets up connection to the Pod's namespace
7. Traffic is forwarded between local and Pod ports

Implementation details:

- CRI `PortForward()` returns a URL for the streaming server
- Kubelet connects to this URL to establish streams
- Multiple ports can be forwarded simultaneously
- Each port uses a separate stream in the multiplexed connection

### 10.7 Resource Management and cgroups

Kubernetes uses cgroups for resource management:

#### 10.7.1 Resource Limits Flow

When Pod resource limits are specified:

1. Kubelet includes resource limits in CRI `ContainerConfig`
    
2. CRI runtime translates these into cgroup settings
    
3. Implementation depends on the cgroup version:
    
    - **cgroupv1**:
        
        - CPU limits: `cpu.cfs_quota_us` and `cpu.cfs_period_us`
        - CPU requests: `cpu.shares`
        - Memory limits: `memory.limit_in_bytes`
    - **cgroupv2**:
        
        - CPU limits: `cpu.max`
        - CPU requests: `cpu.weight`
        - Memory limits: `memory.max`
4. For Pod-level cgroups:
    
    - Resources are divided among containers
    - Overhead is accounted for
    - CPU and memory are managed separately
    - QoS class affects cgroup hierarchy

#### 10.7.2 QoS Classes and cgroups

The kubelet organizes cgroups based on QoS class:

1. **Guaranteed**:
    
    - Requests equal limits for all resources
    - Highest priority during resource contention
    - Placed in their own cgroups with strict limits
2. **Burstable**:
    
    - At least one container has a request
    - Can exceed requests up to limits
    - Middle priority during resource contention
    - Organized in cgroups with reservation but possible bursting
3. **BestEffort**:
    
    - No requests or limits specified
    - Lowest priority during resource contention
    - First to be terminated during resource pressure
    - Minimal cgroup restrictions

#### 10.7.3 CPU Management Policies

Kubelet supports specialized CPU management:

1. **none** (default):
    
    - No CPU pinning
    - Containers can use any CPU
    - cpusets are not used
2. **static**:
    
    - Guaranteed Pods get exclusive CPUs
    - CPUs are pinned using cpusets
    - Remaining CPUs form a shared pool for Burstable and BestEffort Pods
    - Enables better performance for latency-sensitive workloads

Implementation:

- Kubelet maintains internal CPU manager state
- CPU assignments are passed to CRI as cpuset restrictions
- CRI runtime applies these as cgroup cpuset.cpus

### 10.8 Customizing the Kubelet-CRI Interaction

#### 10.8.1 Custom Annotations

Kubernetes allows passing custom information to the CRI:

1. Add annotations to Pod with specific prefixes
2. Kubelet passes these annotations to the CRI
3. CRI implementation can use them for custom behavior

Example use cases:

- GPU configuration
- Custom seccomp profiles
- Runtime-specific optimizations
- Low-level OS tuning

#### 10.8.2 Device Plugins

Device plugins enable support for specialized hardware:

1. Device plugin registers with kubelet
2. Device plugin reports available devices
3. Scheduler considers available devices
4. When Pod is scheduled:
    - Kubelet allocates devices
    - Device plugin sets up devices
    - Kubelet passes device information to CRI
    - CRI runtime makes devices available to containers

Implementation:

- Device plugins communicate with kubelet via Unix sockets
- Kubelet aggregates device information from all plugins
- Device allocations are passed to CRI in `ContainerConfig.devices`
- CRI runtime sets up device access (mknod, bind-mount, etc.)

## 11. The Complete Picture: From kubectl to Container Execution

Let's follow the complete flow of a Pod creation from kubectl to running containers:

### 11.1 User Command and Client-Side Processing

A user runs:

```
kubectl apply -f pod.yaml
```

kubectl performs:

1. Reads and validates the YAML file
2. Converts YAML to JSON
3. Reads kubeconfig for authentication details
4. Creates REST request to the API server
5. Authenticates using client certificates, tokens, or other mechanisms
6. Sends POST request to `/api/v1/namespaces/default/pods`

### 11.2 API Server Processing

The API server:

1. Authenticates the request using one or more authenticators
    
2. Authorizes the operation using RBAC or other authorization modes
    
3. Passes the request through admission controllers:
    
    - **MutatingAdmissionWebhook**: Can modify the request
    - **DefaultStorageClass**: Sets default storage class if needed
    - **LimitRanger**: Applies default resource limits if needed
    - **ServiceAccount**: Ensures service account exists
    - **ValidatingAdmissionWebhook**: Validates the request
    - Many others
4. Validates the Pod object against the schema
    
5. Generates a UID for the Pod
    
6. Sets initial fields (creationTimestamp, status.phase=Pending)
    
7. Persists the Pod object to etcd
    
8. Sends a successful response to kubectl
    

### 11.3 Controller and Scheduler Processing

1. The scheduler watches for Pods without nodeName:
    
    - Retrieves the unscheduled Pod from the API server
    - Filters nodes based on Pod requirements
    - Scores remaining nodes based on affinity, anti-affinity, etc.
    - Selects the best node (e.g., Node1)
    - Updates the Pod with `nodeName: Node1`
    - Persists the update via the API server
2. If the Pod is created by a controller (Deployment, StatefulSet):
    
    - Controller sets owner references
    - Controller adds controller-specific labels
    - Controller watches the Pod for status updates

### 11.4 Kubelet Pod Processing

The kubelet on Node1:

1. Watches for Pods assigned to its node
    
2. Sees the new Pod with `nodeName: Node1`
    
3. Retrieves the full Pod specification
    
4. Admits the Pod through internal admission handlers
    
5. Creates a Pod directory in `/var/lib/kubelet/pods/<pod-uid>/`
    
6. Coordinates with the volume manager to set up volumes:
    
    - PersistentVolumes are attached if needed
    - EmptyDir volumes are created locally
    - Secret and ConfigMap volumes are prepared
7. Constructs a `PodSandboxConfig` with:
    
    - Metadata (name, namespace, UID, etc.)
    - Labels and annotations
    - DNS configuration
    - Network configuration
    - Linux-specific options (namespace, cgroups)
8. Calls RuntimeService `RunPodSandbox()` with this configuration
    
9. CRI runtime creates the Pod sandbox and sets up networking
    
10. Pod moves to status phase "Running"
    

### 11.5 Container Creation via CRI

For each container in the Pod:

1. Kubelet checks if the image is available via `ImageStatus()`
    
2. If not, kubelet calls `PullImage()`
    
3. CRI runtime connects to the registry, authenticates, and pulls the image
    
4. Kubelet constructs a `ContainerConfig` with:
    
    - Image reference
    - Commands and arguments
    - Environment variables
    - Mounts and devices
    - Resource limits
    - Security context
5. Kubelet calls `CreateContainer()` with this configuration
    
6. CRI runtime creates the container but doesn't start it
    
7. Kubelet calls `StartContainer()` to start the container
    
8. CRI runtime starts the container process
    

### 11.6 Container Runtime Operations

The CRI runtime (containerd or CRI-O):

1. Creates OCI runtime bundles with:
    
    - config.json (OCI configuration)
    - rootfs (container filesystem)
2. Invokes the low-level runtime (runc):
    
    - Creates namespaces (if not shared from Pod sandbox)
    - Sets up cgroups according to resource limits
    - Mounts volumes and filesystems
    - Configures devices
    - Applies security settings
    - Executes the container entrypoint
    - Redirects stdio streams
3. Container monitoring process (containerd-shim or conmon):
    
    - Maintains connection to container
    - Buffers container logs
    - Handles container exit

### 11.7 Ongoing Management

Throughout the Pod's lifecycle:

1. Kubelet periodically calls:
    
    - `ListPodSandbox()` and `ListContainers()` for overall state
    - `PodSandboxStatus()` and `ContainerStatus()` for detailed status
    - `ContainerStats()` for resource usage
2. Kubelet performs health checks:
    
    - Liveness probes to determine if container is alive
    - Readiness probes to determine if container can accept traffic
    - Startup probes to determine if application has started
3. Kubelet reports status to the API server:
    
    - Overall Pod phase
    - Individual container statuses
    - Conditions (PodReady, ContainersReady, etc.)
    - Resource usage via the Metrics API
4. Controllers react to status changes:
    
    - ReadinessGates affect Service traffic routing
    - Failed liveness probes trigger restarts
    - Pod status affects Deployment rollout status

## 12. Conclusion: The Kubernetes Container Orchestration Symphony

The Kubernetes architecture represents a sophisticated orchestration system with well-defined interfaces between components. The CRI in particular demonstrates Kubernetes' commitment to extensibility and modularity.

From a high-level request to low-level container execution, the system functions through:

1. **Declarative API**: Users define desired state, not procedures
2. **Control Loops**: Controllers continuously reconcile actual state with desired state
3. **Layered Architecture**: Clean separation of concerns between components
4. **Well-defined Interfaces**: CRI, CNI, CSI enable pluggable implementations
5. **Distributed System Design**: Components are loosely coupled and independently scalable

This architecture enables Kubernetes to manage containerized applications at scale across diverse environments, from developer laptops to massive production clusters, while maintaining a consistent application deployment and management experience.

The kubelet and CRI form a critical bridge between Kubernetes' abstract API resources and the actual containers running on nodes. The detailed interaction between these components ensures that container lifecycles are managed correctly while allowing flexibility in the underlying container runtime technology.