

### 1. **Kubernetes and Distributed Applications:**

Kubernetes is a platform designed to facilitate the creation, deployment, and management of distributed applications. These applications consist of one or more programs running on separate machines. These programs typically accept input, manipulate data, and return results.

Before deploying distributed systems, one of the initial steps is to consider how to build and manage the **container images** that will contain the necessary software components of these programs.

### 2. **Application Dependencies:**

Application programs are typically made up of:

- **Language runtimes** (e.g., Java, Python)
- **Libraries** (both internal and external)
- **Source code**

Many applications rely on external shared libraries, such as `libc` or `libssl`, which are often shipped as components of the OS. These libraries are essential for the program’s operation and are expected to be available on the machine where the program runs.

**Dependency Problems:**

- **Environment discrepancies:** Applications developed in one environment (like a developer’s laptop) might not work properly in a production environment due to missing shared libraries or incompatible versions.
- **Deployment errors:** Even when both development and production environments use the same OS version, issues can arise if developers forget to include necessary assets when deploying the application to production.
- **Coupling between teams:** If different teams are developing separate programs that rely on the same shared libraries, this can cause unwanted complexity and tight coupling between their work.

### 3. **Challenges with Traditional Deployment:**

Traditional methods of deploying applications often involve running **imperative scripts**. This approach can become very complex, especially in distributed systems, leading to difficult and error-prone deployments. Rollouts of new versions of applications can be labor-intensive and complicated.

The solution to many of these challenges lies in **immutable infrastructure** and **immutable images**. These allow for consistent deployment environments where dependencies and configurations are bundled together, ensuring reliable deployment without the need for complex manual scripting.

### 4. **Immutable Images and Containers:**

The key to solving dependency management and deployment issues is the use of **container images**. Container images are immutable, self-contained units that include the application and all of its dependencies. This eliminates discrepancies between environments and reduces errors during deployment.

### 5. **Docker and Containerization:**

Docker is the most widely used tool for creating and running containers. It makes it easy to package applications into **container images**, which can be pushed to a **container registry** (such as Docker Hub or cloud providers’ registries). These images are then shared and can be pulled and run on any machine that supports Docker.

**Container Registries**:

- Registries allow for the management, storage, and sharing of container images.
- Public cloud providers offer container registry services.
- Private container registries can be set up for proprietary images.
- These registries integrate easily with **continuous delivery systems**, automating the process of building and deploying containerized applications.

### 6. **Docker Image Format:**

The Docker image format is the most commonly used container image format and is made up of multiple **layers**. Each layer:

- Adds, removes, or modifies files in the filesystem of the image.
- Represents an incremental change to the previous layer, making it possible to reuse parts of the image across multiple images.

This layering mechanism forms an **overlay filesystem**, which is used both when the image is built and when it is running. Common implementations of such filesystems include `aufs`, `overlay`, and `overlay2`.

### 7. **Open Container Initiative (OCI):**

The **OCI** is a project that aims to standardize container image formats. The Docker image format has been aligned with the OCI image format standard. While Docker continues to be the dominant format, OCI compatibility ensures broader support across different container runtimes, including Kubernetes.

### 8. **The Workflow of Using Docker Containers:**

- **Creating a Container Image:** To work with a containerized application, you first create a Docker image, which packages the application and its dependencies into a single file.
- **Running the Application:** Once the Docker image is available, it can be run using Docker’s container runtime. The containerized application will execute within an isolated environment, independent of the host machine's configuration.

### 9. **Summary of Topics in the Chapter:**

The chapter introduces the process of packaging and running applications using Docker containers:

- **Packaging an application** with the Docker image format.
- **Running the application** using Docker’s container runtime.


## Understanding container layering

The concept of **container layering** is central to how Docker and container images are constructed and stored

### **1. Understanding Container Images:**

- A **container image** is not a single file but rather a specification defined by a **manifest file**. This manifest points to other files, which are the actual layers that make up the container image.
- The **manifest** and associated layers are often treated together as a unit, allowing for a more efficient method of storing and transmitting container images. This approach is referred to as **layered architecture**.
- There is an **API** that facilitates uploading and downloading images to and from a **container image registry** (like Docker Hub, AWS ECR, etc.).

### **2. Image Construction with Filesystem Layers:**

- **Container images** are constructed as a stack of filesystem **layers**. Each layer inherits from the previous one, meaning that modifications are made on top of existing files.
- The ordering of these layers is important, as each layer **builds upon** the layers that came before it. However, the example here reverses the ordering for simplicity.

### **3. Example of Container Layering:**

Let’s walk through an example to demonstrate how this layering works:

- **Container A:** This is the **base container**, and it contains only the operating system files, such as Debian.
    
    csharp
    
    `└── container A: base OS (e.g., Debian)`
    
- **Container B:** This builds upon **Container A**, adding **Ruby v2.1.10**. At this point, Container B has all the files from Container A, plus the Ruby installation.
    
    csharp

    `└── container B: base OS (Debian) + Ruby v2.1.10`
    
- **Container C:** This also builds upon **Container A**, but instead of Ruby, it adds **Golang v1.6**. Therefore, Container C shares the base OS files from A but includes Golang.
    
    csharp

    
    `└── container C: base OS (Debian) + Golang v1.6`
    

Now, we have three containers:

- **A** (base OS),
- **B** (Ruby on top of A),
- **C** (Golang on top of A).

Containers **B** and **C** share nothing except the base operating system files from **A**.

### **4. Adding More Layers:**

We can continue to build upon these containers by adding additional software or modifications, such as web frameworks.

- **Container D:** We can build **Container D** on top of **Container B** by adding **Rails v4.2.6**. This would make **D** a container with **Debian + Ruby v2.1.10 + Rails v4.2.6**.
    
    csharp
    
    `└── container D: base OS (Debian) + Ruby v2.1.10 + Rails v4.2.6`
    
- **Container E:** We might also need to support an older application that requires a legacy version of Ruby on Rails (e.g., version 3.2.x). In this case, **Container E** would also be based on **Container B** but with **Rails v3.2.x** instead of the newer version.
    
    csharp
    
    `└── container E: base OS (Debian) + Ruby v2.1.10 + Rails v3.2.x`
    

Now we have:

- **B** (Ruby),
- **D** (Ruby + Rails 4.2.6),
- **E** (Ruby + Rails 3.2.x).

### **5. Directed Acyclic Graph (DAG) of Containers:**

Conceptually, each layer in a container image is built upon its parent image. This parent-child relationship forms a **directed acyclic graph (DAG)**, meaning that images can be chained together in a hierarchy, with each image inheriting from its predecessor.

- In this example, **Container D** and **Container E** both share the same base, **Container B**, and modify it in different ways by adding different versions of Rails.
- The layers create a graph-like structure that can be extended with many more images and layers, allowing for a highly flexible and efficient image system.

### **6. Efficiency of Layered Container Images:**

- **Layer Reusability:** One of the primary benefits of this layering model is the reuse of common layers. For example, if multiple containers share the same base operating system (e.g., **Debian**), that layer is only stored once. This reduces storage requirements and speeds up image transfers.
- **Efficient Distribution:** When a new container image is pushed to a registry, only the layers that are new or changed need to be uploaded. Existing layers can be shared across multiple images, making distribution faster and more efficient.

### **7. Container Configuration File:**

Along with the **container image**, there is typically a **container configuration file** that provides crucial instructions on how the container environment should be set up. This configuration file plays a key role in defining how the container will operate at runtime and how it should interact with the underlying host system. Here are some key components of the configuration:

- **Entry Point:** Defines the command or executable that should run when the container starts. It specifies the entry point for the containerized application.
- **Networking:** Defines the network settings for the container, including which ports to expose, networking modes (e.g., bridge, host), and how the container communicates with other containers or external services.
- **Namespace Isolation:** Specifies the isolation level between containers, which is achieved by using namespaces in Linux (e.g., PID namespace, network namespace, etc.). This isolation ensures that each container operates in its own environment, separate from others.
- **Resource Constraints (cgroups):** Resource limitations, such as CPU and memory restrictions, are set using **cgroups** (Control Groups) to prevent containers from consuming excessive resources and affecting the host or other containers.
- **Syscall Restrictions:** Specifies which system calls (syscalls) should be allowed or restricted in a container. This is important for security, as some syscalls can be used by malicious actors to break out of the container environment.

The **Docker image format** bundles the root filesystem and configuration file together. This ensures that when a container is launched, it has all the necessary setup and instructions to execute the application correctly and securely.

### **9. Types of Containers:**

Containers generally fall into two main categories: **System Containers** and **Application Containers**.

#### **System Containers:**

- **System containers** are designed to mimic **virtual machines (VMs)** and typically run a full boot process, much like a traditional OS. They include a variety of system services commonly found in VMs, such as:
    - **SSH** (for remote access)
    - **Cron** (for scheduled tasks)
    - **Syslog** (for logging)
- In the early days of Docker, system containers were more common, as they allowed users to run an entire OS in a container.
- However, **system containers** have become **less popular** over time due to their complexity and resource overhead. They are often seen as **poor practice** in containerized environments because they do not fully leverage the lightweight and efficient nature of containers.

#### **Application Containers:**

- **Application containers**, in contrast, focus on running a **single program** or service within a container. This simple approach allows for better **scalability** and **modularity**.
- The idea of running just one application per container is central to **microservices** architecture and the **12-factor app** methodology, which advocates that applications should be designed to run in isolated, independent containers for better flexibility and scalability.
- While this might seem like a constraint, it actually provides a level of granularity that is perfect for composing complex, **scalable applications**. Each container is a single unit of functionality, making it easier to manage, scale, and update.
- This design philosophy aligns with the concept of **Pods** in Kubernetes, which allow multiple related containers to run together within a shared context (such as shared networking, storage, and lifecycle management).

### **9. Why Application Containers are Preferred:**

Application containers are preferred over system containers for several reasons:

- **Lightweight:** Since they only run a single application or service, they are more efficient and consume fewer resources compared to system containers, which run a full OS and multiple system services.
- **Scalable:** The modular nature of application containers makes them highly scalable. You can easily replicate or scale the containerized service to meet demand.
- **Focused Isolation:** By isolating applications into their own containers, you can avoid conflicts and dependency issues, ensuring that each service runs in its optimal environment without interference from other services.
- **DevOps and CI/CD Friendly:** The single application per container model is well-suited for **DevOps workflows**, **CI/CD pipelines**, and **microservices architectures**, where each service needs to be independently developed, deployed, and managed.

### **10. Kubernetes and Pods:**

In **Kubernetes**, containers are often grouped together into **Pods**. A Pod is the smallest deployable unit in Kubernetes and can contain one or more containers. While each container within a Pod runs its own application, they share:

- **Network namespaces:** They can communicate with each other via `localhost`.
- **Storage volumes:** Containers within a Pod can share storage.
- **Lifecycle:** All containers in a Pod are managed together and are typically started and stopped together.

The Pod model reflects the philosophy of application containers, where each container focuses on running a specific part of the application, and Pods allow related containers to be managed as a single entity, ensuring that they work together seamlessly.

---


### **Storing Images in a Remote Registry**

A container image is useful only if it is accessible across all machines in the cluster, and this is especially true for Kubernetes, which relies on accessing images from a central registry. Rather than manually exporting and importing images to each machine—which can be error-prone and tedious—the **Docker community standard** is to store container images in a **remote registry**.

#### **Types of Docker Registries**

Docker registries can either be **public** or **private**, and the choice depends on your specific needs.

- **Public Registries**: These allow anyone to download images and are great for sharing software with the world. Public registries, like **Docker Hub**, allow for easy, unauthenticated access to images. These are ideal for widely distributed software where you want users everywhere to have the same experience.
    
- **Private Registries**: These require authentication to download images and are best for storing proprietary or internal applications that should not be exposed to the public. Private registries also offer better **availability** and **security guarantees** tailored to your organization.
    

#### **Storing Docker Images in a Remote Registry**

To push an image to a registry, you need to authenticate using the `docker login` command. While this process is similar across registries, we will use **Google Container Registry (GCR)** as an example in this book. Other cloud platforms, like **AWS** and **Azure**, have their own container registries.

Once authenticated, you can tag an image with the registry’s URL and an optional version or variant identifier:

bash

`$ docker tag kuard gcr.io/kuar-demo/kuard-amd64:blue`

This tags the `kuard` image and prepares it for upload. Next, you can push the image to the remote registry:

bash

`$ docker push gcr.io/kuar-demo/kuard-amd64:blue`

Now that the image is available in the remote registry, it can be accessed globally. Since the image was pushed as **public**, it will be available to anyone without authentication.

---

### **The Container Runtime Interface (CRI)**

Kubernetes relies on container runtimes to run containers using the specific container APIs of the target operating system. For Linux, this involves setting up **cgroups** and **namespaces**. The **Container Runtime Interface (CRI)** is the API standard Kubernetes uses to interact with container runtimes.

Several container runtimes implement the CRI, including:

- **containerd-cri**: Provided by Docker.
- **cri-o**: A runtime developed by Red Hat.

Since Kubernetes 1.25, only container runtimes that support the **CRI** are compatible with Kubernetes. Fortunately, most **managed Kubernetes** providers have made this transition seamless for users.

---

### **Running Containers with Docker**

While Kubernetes uses the **kubelet** daemon on each node to launch containers, the **Docker CLI tool** is often easier to use for local development or testing. To run a container from a remote image (e.g., `gcr.io/kuar-demo/kuard-amd64:blue`), use the following Docker command:

bash

`$ docker run -d --name kuard \ --publish 8080:8080 \ gcr.io/kuar-demo/kuard-amd64:blue`

#### **Explanation:**

- **-d**: Run the container in the background (detached mode).
- **--name kuard**: Assign a friendly name to the container.
- **--publish 8080:8080**: Map port 8080 on your local machine to port 8080 in the container. This is necessary because containers get their own IP addresses, and without port forwarding, your local machine cannot access the container’s ports.

By running this command, the **kuard** container will start and be accessible on `http://localhost:8080` in the browser.

### **Limiting Resource Usage in Docker**

Docker allows you to control the resource usage of containers, leveraging Linux **cgroup** technology to limit memory, CPU, and swap usage. Kubernetes also uses this feature to restrict the resources each **Pod** consumes. This ensures fair usage of resources across multiple applications running on the same hardware.

---

### **Limiting Memory Resources**

One of the main advantages of running applications inside containers is the ability to restrict memory usage. This ensures that containers do not consume excessive memory, allowing multiple applications to coexist and ensuring fairness.

To limit memory usage, you can use the `--memory` and `--memory-swap` flags when starting a container. For example, to limit the **kuard** container to 200 MB of memory and 1 GB of swap space, use the following steps:

1. **Stop and remove the existing container:**
    
    bash
    
    `$ docker stop kuard $ docker rm kuard`
    
2. **Start a new container with memory limits:**
    
    bash
    
    
    `$ docker run -d --name kuard \ --publish 8080:8080 \ --memory 200m \ --memory-swap 1G \ gcr.io/kuar-demo/kuard-amd64:blue`
    
    - **--memory 200m**: Limits the container to 200 MB of memory.
    - **--memory-swap 1G**: Allows the container to use 1 GB of swap space.

If the program inside the container exceeds the memory limit, it will be **terminated** automatically by the system.

---

### **Limiting CPU Resources**

CPU resources are another critical component that can be limited to ensure fair usage among containers. The `--cpu-shares` flag allows you to control the CPU share assigned to the container.

For example, to limit CPU usage for the **kuard** container, use the following command:

bash


`$ docker run -d --name kuard \ --publish 8080:8080 \ --memory 200m \ --memory-swap 1G \ --cpu-shares 1024 \ gcr.io/kuar-demo/kuard-amd64:blue`

- **--cpu-shares 1024**: Sets the CPU weight for the container. The default is 1024, which means the container gets an equal share of CPU resources compared to other containers with the same value. You can adjust this value to give the container more or less CPU share relative to others.

---

### **Summary:**

1. **Memory Limits**: Use `--memory` and `--memory-swap` to restrict memory and swap usage in Docker containers.
2. **CPU Limits**: Use `--cpu-shares` to control the CPU share allocated to a container, helping manage resource usage in a multi-container environment.
3. **Resource Fairness**: These limitations ensure that containers don't exhaust system resources, allowing fair distribution of memory, CPU, and swap space.