# 📘 Kubernetes RBAC: Roles, ClusterRoles, Bindings & Aggregation (With Real-World Scenarios)

## 🔐 Introduction

Kubernetes uses **Role-Based Access Control (RBAC)** to define who can access what within a cluster. It controls access to resources such as Pods, Services, Secrets, Nodes, and more.

This document breaks down key RBAC components:

- `Role`
    
- `ClusterRole`
    
- `RoleBinding`
    
- `ClusterRoleBinding`
    
- Aggregated `ClusterRoles`
    

Each section includes **real-world scenarios** for practical understanding.

## 📊 RBAC Components Visualization

```mermaid
graph TB
    subgraph "Core Components"
        Role["Role<br>(Namespace-scoped)"]
        CR["ClusterRole<br>(Cluster-wide)"]
        RB["RoleBinding<br>(Namespace-scoped)"]
        CRB["ClusterRoleBinding<br>(Cluster-wide)"]
    end
    
    subgraph "Subjects"
        User[User]
        Group[Group]
        SA[ServiceAccount]
    end
    
    Role -->|can be bound by| RB
    CR -->|can be bound by| RB
    CR -->|can be bound by| CRB
    
    RB -->|grants permissions to| User
    RB -->|grants permissions to| Group
    RB -->|grants permissions to| SA
    CRB -->|grants permissions to| User
    CRB -->|grants permissions to| Group
    CRB -->|grants permissions to| SA
    
    style Role fill:#f9f,stroke:#333,stroke-width:2px
    style CR fill:#bbf,stroke:#333,stroke-width:2px
    style RB fill:#ff9,stroke:#333,stroke-width:2px
    style CRB fill:#9f9,stroke:#333,stroke-width:2px
```

---

## 🎯 Role

**Scope**: Namespace-specific  
**Purpose**: Grant access to resources within a single namespace

### Use Case: Application Developer in a Dev Namespace

A team of developers is working in the `dev` namespace. You want them to view and troubleshoot deployments but not modify anything outside of that scope. You create a Role to allow access to list, get, and watch Pods and Deployments in the `dev` namespace.

This ensures:

- Developers stay limited to their environment
    
- They cannot accidentally affect production or other namespaces
    
- It adheres to the principle of least privilege
    

```mermaid
graph TD
    subgraph "dev namespace"
        Role["Role:<br>app-developer"]
        RB["RoleBinding:<br>dev-team-access"]
        Pod[Pods]
        Deploy[Deployments]
        Service[Services]
        
        Role -->|defines permissions| Perms["Permissions:<br>- get pods<br>- list pods<br>- watch pods<br>- get deployments<br>- list deployments<br>- watch deployments"]
        Role --- RB
        RB -->|binds to| DevTeam["Developer Group"]
        
        Perms -->|allows access to| Pod
        Perms -->|allows access to| Deploy
    end
    
    subgraph "prod namespace"
        ProdPod[Pods]
        ProdDeploy[Deployments]
    end
    
    DevTeam -.-x ProdPod
    DevTeam -.-x ProdDeploy
    
    style Role fill:#f9f,stroke:#333,stroke-width:2px
    style RB fill:#ff9,stroke:#333,stroke-width:2px
    style DevTeam fill:#bbf,stroke:#333,stroke-width:2px
```

---

## 🌐 ClusterRole

**Scope**: Cluster-wide  
**Purpose**: Grant access across all namespaces or to non-namespaced resources

### Use Case: Monitoring System or Cluster Admin

A monitoring tool like Prometheus needs to scrape metrics from workloads across all namespaces. Since it operates at a global level, it requires permissions beyond a single namespace.

Alternatively, your cluster admin team needs access to resources like Nodes and PersistentVolumes that are not confined to any namespace. A ClusterRole provides this access.

This enables:

- Cross-namespace visibility
    
- Management of infrastructure-level resources
    
- Shared roles that multiple namespaces can reuse via bindings
    

```mermaid
graph TB
    subgraph "Cluster-wide Resources"
        CR["ClusterRole:<br>monitoring-metrics"]
        CRB["ClusterRoleBinding:<br>prometheus-access"]
        Nodes[Nodes]
        PV[PersistentVolumes]
    end
    
    subgraph "Namespace A"
        PodsA[Pods]
        DeployA[Deployments]
    end
    
    subgraph "Namespace B"
        PodsB[Pods]
        DeployB[Deployments]
    end
    
    CR -->|defines permissions| Perms["Permissions:<br>- get pods<br>- list pods<br>- get metrics<br>- get nodes"]
    CR --- CRB
    CRB -->|binds to| Prometheus["Prometheus ServiceAccount"]
    
    Perms -->|allows access to| Nodes
    Perms -->|allows access to| PodsA
    Perms -->|allows access to| PodsB
    
    style CR fill:#bbf,stroke:#333,stroke-width:2px
    style CRB fill:#9f9,stroke:#333,stroke-width:2px
    style Prometheus fill:#d9f,stroke:#333,stroke-width:2px
```

---

## 🔗 RoleBinding

**Scope**: Namespace-specific  
**Purpose**: Assign a Role (or ClusterRole) to a user, group, or service account within a namespace

### Use Case: Service Account for CI/CD Job

A CI/CD pipeline running in the `staging` namespace needs to deploy applications and manage resources there. Using a RoleBinding, you can assign a Role that allows deploying apps, restarting Pods, and managing Services — **but only within `staging`**.

This:

- Keeps the service account sandboxed to its own namespace
    
- Prevents security risks from over-privileging
    
- Encourages separation of duties
    

```mermaid
graph TD
    subgraph "staging namespace"
        Role["Role:<br>deployer"]
        RB["RoleBinding:<br>cicd-deployer"]
        Pod[Pods]
        Deploy[Deployments]
        Service[Services]
        
        Role -->|defines permissions| Perms["Permissions:<br>- create/update deployments<br>- create/update services<br>- get/list pods<br>- delete pods"]
        Role --- RB
        RB -->|binds to| CICD["CI/CD ServiceAccount"]
        
        Perms -->|allows access to| Pod
        Perms -->|allows access to| Deploy
        Perms -->|allows access to| Service
        CICD -->|deploys to| staging["staging resources"]
    end
    
    subgraph "prod namespace"
        ProdDeploy[Deployments]
    end
    
    CICD -.-x ProdDeploy
    
    style Role fill:#f9f,stroke:#333,stroke-width:2px
    style RB fill:#ff9,stroke:#333,stroke-width:2px
    style CICD fill:#bbf,stroke:#333,stroke-width:2px
```

---

## 🌍 ClusterRoleBinding

**Scope**: Cluster-wide  
**Purpose**: Assign a ClusterRole to a user, group, or service account across the entire cluster

### Use Case: SRE Team with Global Access

Your site reliability engineers (SREs) need to troubleshoot incidents in any namespace, review logs, and update deployments. You bind a ClusterRole with these privileges to their group using a ClusterRoleBinding.

This provides:

- Uniform access across all namespaces
    
- Simpler administration by avoiding namespace-specific bindings
    
- Efficient onboarding/offboarding by managing access at group level
    

```mermaid
graph TB
    subgraph "Cluster Level"
        CR["ClusterRole:<br>sre-admin"]
        CRB["ClusterRoleBinding:<br>sre-team-access"]
        Nodes[Nodes]
        NS[Namespaces]
    end
    
    subgraph "Any Namespace"
        Pods[Pods]
        Deploys[Deployments]
        Logs["Pod Logs"]
        Config[ConfigMaps]
    end
    
    CR -->|defines permissions| Perms["Permissions:<br>- get/list/update all resources<br>- get logs<br>- exec into pods<br>- manage nodes"]
    CR --- CRB
    CRB -->|binds to| SRE["SRE Team Group"]
    
    Perms -->|allows access to| Nodes
    Perms -->|allows access to| NS
    Perms -->|allows access to| Pods
    Perms -->|allows access to| Deploys
    Perms -->|allows access to| Logs
    Perms -->|allows access to| Config
    
    style CR fill:#bbf,stroke:#333,stroke-width:2px
    style CRB fill:#9f9,stroke:#333,stroke-width:2px
    style SRE fill:#d9f,stroke:#333,stroke-width:2px
```

---

## 🧩 Aggregated ClusterRoles

**Scope**: Cluster-wide (composed from other ClusterRoles)  
**Purpose**: Dynamically combine multiple `ClusterRoles` into one via label-based selectors

### Use Case: Building a Custom Read-Only Role

You want to create a custom "read-only observer" role that includes:

- Viewing core resources (Pods, Services)
    
- Viewing logs
    
- Accessing metrics
    

Instead of creating a large, monolithic ClusterRole, you define small ClusterRoles for each of these and label them consistently. Then, you define an aggregated ClusterRole that automatically pulls in rules from those smaller ones.

This approach:

- Promotes modularity and reusability
    
- Makes it easy to update access by adjusting labeled roles
    
- Ensures consistency across environments and teams
    

```mermaid
graph LR
    subgraph "Aggregation Components"
        CR1["ClusterRole: base-viewer<br>label: aggregate-to-observer: true<br><br>- get/list pods<br>- get/list services"]
        CR2["ClusterRole: logs-viewer<br>label: aggregate-to-observer: true<br><br>- get pods/log"]
        CR3["ClusterRole: metrics-viewer<br>label: aggregate-to-observer: true<br><br>- get pods/metrics"]
        
        AGG["Aggregated ClusterRole: observer<br><br>aggregationRule:<br>  clusterRoleSelectors:<br>  - matchLabels:<br>      aggregate-to-observer: true"]
    end
    
    CR1 -->|aggregated into| AGG
    CR2 -->|aggregated into| AGG
    CR3 -->|aggregated into| AGG
    
    AGG -->|results in| Result["Effective Permissions:<br>- get/list pods<br>- get/list services<br>- get pods/log<br>- get pods/metrics"]
    
    style CR1 fill:#bbf,stroke:#333,stroke-width:2px
    style CR2 fill:#bbf,stroke:#333,stroke-width:2px
    style CR3 fill:#bbf,stroke:#333,stroke-width:2px
    style AGG fill:#d9f,stroke:#333,stroke-width:2px
    style Result fill:#9f9,stroke:#333,stroke-width:2px
```

---

## 📊 RBAC Summary Table

|Type|Scope|Resources|Bound With|Use Case Scenario|
|---|---|---|---|---|
|`Role`|Namespace|Namespaced only|`RoleBinding`|Developers managing apps within one namespace|
|`ClusterRole`|Cluster-wide|All or non-namespaced|`RoleBinding`, `ClusterRoleBinding`|SREs, monitoring tools, and infrastructure access|
|`Aggregated ClusterRole`|Cluster-wide|Dynamically composed|Same as `ClusterRole`|Custom composite roles with reusable permissions|
|`RoleBinding`|Namespace|-|Binds Role or ClusterRole|CI/CD pipelines restricted to namespace resources|
|`ClusterRoleBinding`|Cluster-wide|-|Binds ClusterRole|Admin groups or tools needing access across cluster|

---

### Introduction to the RBAC Resource Scope Table

The table below provides a **concise summary of how Kubernetes RBAC rules apply to different resource types**, distinguishing between **namespaced** and **non-namespaced** resources. It explains:

- **When to use `Role` vs `ClusterRole`**
    
- **How to scope permissions using `RoleBinding` or `ClusterRoleBinding`**
    
- **Which Kubernetes resources require cluster-level access**
    

## Summary: RBAC Rule Application in Kubernetes

|Rule Type|Applies To|Scope Type|Resource Examples|Role Type Required|Notes|
|---|---|---|---|---|---|
|`Role`|**Namespaced only**|Namespaced|`pods`, `services`, `deployments`, `configmaps`, `secrets`|`Role`|Must be created in a specific namespace and used with `RoleBinding`|
|`ClusterRole`|**All namespaces**|Namespaced + Cluster-wide|All of the above plus `nodes`, `persistentvolumes`, `namespaces`|`ClusterRole`|Can be bound cluster-wide or per-namespace via `RoleBinding`|
|`RoleBinding`|Binds to a **Role or ClusterRole**|Namespaced|-|n/a|Grants access **within one namespace only**|
|`ClusterRoleBinding`|Binds to **ClusterRole**|Cluster-wide|-|n/a|Grants access **across all namespaces** and for cluster-scoped resources|

---

## 🔍 Examples by Resource Type

|Resource|API Group|Namespaced?|Needs Role or ClusterRole|Notes|
|---|---|---|---|---|
|`pods`|`""` (core)|✅ Yes|`Role` or `ClusterRole`|Most used in workload RBAC|
|`services`|`""` (core)|✅ Yes|`Role` or `ClusterRole`||
|`deployments`|`apps`|✅ Yes|`Role` or `ClusterRole`||
|`configmaps`|`""`|✅ Yes|`Role` or `ClusterRole`||
|`secrets`|`""`|✅ Yes|`Role` or `ClusterRole`|Sensitive—often scoped tightly|
|`nodes`|`""`|❌ No|`ClusterRole` only|Node-level access; never namespaced|
|`persistentvolumes`|`""`|❌ No|`ClusterRole` only||
|`namespaces`|`""`|❌ No|`ClusterRole` only|Used for managing namespaces themselves|
|`clusterroles`|`rbac.authorization.k8s.io`|❌ No|`ClusterRole` only|RBAC access to RBAC|

## 🧮 Permission Calculation and Accumulation

Kubernetes RBAC uses a purely additive model where permissions are combined rather than overridden:

```mermaid
graph TB
    subgraph "Permission Calculation Example"
        User["User: jane@example.com"]
        
        Role1["Role in dev namespace:<br>- get/list pods"]
        Role2["Role in dev namespace:<br>- update configmaps"]
        CR["ClusterRole:<br>- read secrets"]
        
        RB1["RoleBinding in dev namespace"]
        RB2["RoleBinding in dev namespace"]
        CRB["ClusterRoleBinding"]
        
        Role1 --- RB1 -->|gives namespace<br>permissions| User
        Role2 --- RB2 -->|gives namespace<br>permissions| User
        CR --- CRB -->|gives cluster-wide<br>permissions| User
        
        User -->|has combined permissions| Final["Effective Permissions:<br><br>In dev namespace:<br>- get/list pods<br>- update configmaps<br>- read secrets<br><br>In all other namespaces:<br>- read secrets only"]
    end
    
    style User fill:#bbf,stroke:#333,stroke-width:2px
    style Final fill:#9f9,stroke:#333,stroke-width:2px
```

---

## 🧠 Quick Guidelines

|Goal|Use This|Notes|
|---|---|---|
|Access only in one namespace|`Role` + `RoleBinding`|Most secure, least privilege|
|Share access across namespaces|`ClusterRole` + `RoleBinding`|Good for shared read-only or logging roles|
|Grant global or infra access|`ClusterRole` + `ClusterRoleBinding`|Needed for nodes, PVs, RBAC, etc.|

---

# Kubernetes RBAC: Understanding Role and Binding Precedence

## 🔐 Introduction

Kubernetes RBAC (Role-Based Access Control) allows fine-grained access control over Kubernetes resources using roles and bindings. One common point of confusion is whether one role or binding can **override**, **restrict**, or **take precedence** over another.

This document explains in detail:

- Whether `ClusterRole` can override a `Role`
    
- Whether `RoleBindings` or `ClusterRoleBindings` can override each other
    
- How Kubernetes RBAC actually computes effective permissions
    
- What mechanisms (if any) can restrict or deny access
    

---

## 🧩 Key Concepts

|Term|Scope|Description|
|---|---|---|
|`Role`|Namespace|Grants access to namespaced resources within a specific namespace|
|`ClusterRole`|Cluster-wide|Grants access to both namespaced and non-namespaced resources|
|`RoleBinding`|Namespace|Binds a `Role` or `ClusterRole` to a subject within a namespace|
|`ClusterRoleBinding`|Cluster-wide|Binds a `ClusterRole` to a subject across the entire cluster|

## 📝 RBAC Additive Nature Visualization

```mermaid
graph TD
    subgraph "RBAC Additive Permission Model"
        SA["ServiceAccount:<br>backend-app"]
        
        subgraph "Roles"
            R1["Role: pod-reader<br>- get pods<br>- list pods"]
            R2["Role: log-reader<br>- get pods/log"]
            CR["ClusterRole: configmap-reader<br>- get configmaps<br>- list configmaps"]
        end
        
        subgraph "Bindings"
            RB1["RoleBinding: pod-reader-binding"]
            RB2["RoleBinding: log-reader-binding"]
            RB3["RoleBinding: configmap-binding"]
        end
        
        R1 --- RB1 -->|binds| SA
        R2 --- RB2 -->|binds| SA
        CR --- RB3 -->|binds| SA
        
        SA -->|has permissions| Final["Effective Permissions:<br>- get pods<br>- list pods<br>- get pods/log<br>- get configmaps<br>- list configmaps"]
    end
    
    style SA fill:#bbf,stroke:#333,stroke-width:2px
    style Final fill:#9f9,stroke:#333,stroke-width:2px
```

---

## ❓ Can a `ClusterRole` Override a `Role`?

**No**, Kubernetes does not support **override behavior** between `ClusterRoles` and `Roles`.

RBAC in Kubernetes is purely **additive**. This means:

- Permissions from all roles and bindings **accumulate**
    
- No role or binding can remove or restrict access granted by another
    
- Kubernetes calculates the **union of all granted permissions**
    

### Example:

Let's say a service account is:

- Bound to a `Role` in namespace `dev` allowing `list` pods
    
- Also bound (via `RoleBinding`) to a `ClusterRole` that allows `get` pods
    

Effective access in `dev`:

- ✅ `list` pods (from Role)
    
- ✅ `get` pods (from ClusterRole)
    

> 🔑 Result: The service account can perform both actions — **no role overrides or limits** the other.

---

## ❓ Can a `RoleBinding` Override a `ClusterRoleBinding`?

**No.** Again, the behavior is additive:

- A `RoleBinding` grants permissions **within a namespace**
    
- A `ClusterRoleBinding` grants permissions **cluster-wide**
    

If a subject is bound by both:

- They receive **combined** permissions in any namespace affected

### Example:

If `alvo-svc` is:

- Bound to a `Role` in namespace `frontend` with permission to `list pods`
    
- Also bound to a `ClusterRole` that grants `get` and `delete pods` via `ClusterRoleBinding`
    

Effective permissions:

- In namespace `frontend`: ✅ `list`, `get`, `delete` pods
    
- In other namespaces: ✅ `get`, `delete` pods (from `ClusterRoleBinding`)
    

> 🔑 No binding "wins" or overrides — **all valid bindings contribute to the final access profile**.

---

## 🚫 What You Cannot Do with RBAC

|Attempted Action|Supported by RBAC?|Explanation|
|---|---|---|
|Deny a specific action|❌ No|RBAC has no `deny` rules; only allows what is explicitly permitted|
|Override or restrict an existing role|❌ No|Roles and bindings cannot subtract or override previously granted access|
|Prioritize one binding over another|❌ No|Kubernetes merges all valid bindings and roles|

## 🔒 RBAC Limitations and Alternatives

```mermaid
graph LR
    subgraph "RBAC Limitation"
        RBAC["Kubernetes RBAC<br>(Additive Only)"]
        NoDeny["No Deny Capability"]
        NoOverride["No Override Mechanism"]
    end
    
    subgraph "Alternative Controls"
        Kyverno["Kyverno<br>Policy Engine"]
        OPA["OPA Gatekeeper<br>Policy Framework"]
        PSA["Pod Security Admission"]
        NetPol["Network Policies"]
    end
    
    RBAC -->|limitation| NoDeny
    RBAC -->|limitation| NoOverride
    
    NoDeny -->|addressed by| Kyverno
    NoDeny -->|addressed by| OPA
    NoOverride -->|addressed by| Kyverno
    NoOverride -->|addressed by| OPA
    NoOverride -->|partially addressed by| PSA
    
    style RBAC fill:#bbf,stroke:#333,stroke-width:2px
    style Kyverno fill:#9f9,stroke:#333,stroke-width:2px
    style OPA fill:#9f9,stroke:#333,stroke-width:2px
```

---

## 🔐 How to Actually Restrict Access

Since RBAC does not support denying access or overriding, you need **complementary tools** to enforce restrictions:

### ✅ Use the following:

|Tool / Feature|Purpose|
|---|---|
|**Kyverno**|Enforce image policies, prevent exec access, etc.|
|**OPA/Gatekeeper**|Rego-based policies for validating and restricting cluster behavior|
|**PodSecurity Standards**|Restrict pod-level behavior like privileged access|
|**NetworkPolicies**|Restrict traffic between Pods|
|**Namespace isolation**|Create strong logical separation and avoid cross-binding|

## 📉 Role Escalation Prevention

```mermaid
graph TD
    subgraph "Role Escalation Prevention"
        Check["Kubernetes API Server"]
        
        Request["Request:<br>Create RoleBinding<br>with elevated permissions"]
        
        Rule["Rule:<br>User can only grant<br>permissions they possess"]
        
        Outcome["Outcome:<br>Request rejected if<br>user lacks permissions<br>they're trying to grant"]
    end
    
    Request -->|submitted to| Check
    Check -->|enforces| Rule
    Rule -->|determines| Outcome
    
    style Check fill:#bbf,stroke:#333,stroke-width:2px
    style Rule fill:#f9f,stroke:#333,stroke-width:2px
    style Outcome fill:#f99,stroke:#333,stroke-width:2px
```

---

## 🧠 Best Practices Summary

|Best Practice|Why It Matters|
|---|---|
|Avoid assigning multiple powerful roles to a single subject|Prevents unintended permission escalation|
|Use `Role` where possible before using `ClusterRole`|Follows least-privilege principle|
|Regularly audit bindings and permissions|Ensure no over-privileged accounts exist|
|Use Kyverno or Gatekeeper for restriction|Compensates for RBAC's additive-only design|

## 🔓 Common RBAC Pitfalls to Avoid

```mermaid
graph TD
    subgraph "Common RBAC Pitfalls"
        P1["Using the default<br>ServiceAccount"]
        P2["Too many<br>cluster-admin bindings"]
        P3["Overlapping roles<br>with different scopes"]
        P4["Relying on RBAC alone<br>for security"]
        
        S1["Create dedicated<br>ServiceAccounts"]
        S2["Implement just-enough<br>admin access"]
        S3["Audit and document<br>role combinations"]
        S4["Add policy engines and<br>admission controllers"]
    end
    
    P1 -->|solution| S1
    P2 -->|solution| S2
    P3 -->|solution| S3
    P4 -->|solution| S4
    
    style P1 fill:#f99,stroke:#333,stroke-width:2px
    style P2 fill:#f99,stroke:#333,stroke-width:2px
    style P3 fill:#f99,stroke:#333,stroke-width:2px
    style P4 fill:#f99,stroke:#333,stroke-width:2px
    style S1 fill:#9f9,stroke:#333,stroke-width:2px
    style S2 fill:#9f9,stroke:#333,stroke-width:2px
    style S3 fill:#9f9,stroke:#333,stroke-width:2px
    style S4 fill:#9f9,stroke:#333,stroke-width:2px
```

---

## ✅ Conclusion

Kubernetes RBAC is designed to **grant** access — not to deny, override, or restrict it. All permissions granted by `Roles`, `ClusterRoles`, `RoleBindings`, and `ClusterRoleBindings` are **cumulative**. This makes it easy to compose access for different use cases, but it also means you must be careful not to over-grant access inadvertently.

> To restrict access or enforce organizational policies, **pair RBAC with tools like Kyverno or OPA Gatekeeper**.

## 🔍 Advanced RBAC Patterns

```mermaid
graph TB
    subgraph "Advanced RBAC Implementation Patterns"
        P1["🔹 Tiered Access Model"]
        P2["🔹 Role Composition"]
        P3["🔹 Namespace Delegation"]
        P4["🔹 Temporary Elevated Access"]
        
        P1 -->|implements| T1["Multiple roles with<br>increasing privileges"]
        P2 -->|implements| T2["Small, focused roles<br>aggregated as needed"]
        P3 -->|implements| T3["Admin rights limited to<br>specific namespaces"]
        P4 -->|implements| T4["Just-in-time access<br>with time constraints"]
    end
    
    style P1 fill:#ddf,stroke:#333,stroke-width:2px
    style P2 fill:#ddf,stroke:#333,stroke-width:2px
    style P3 fill:#ddf,stroke:#333,stroke-width:2px
    style P4 fill:#ddf,stroke:#333,stroke-width:2px
```

## 🛠️ RBAC Troubleshooting Guide

When facing permission issues in Kubernetes, this troubleshooting flow can help identify and resolve the problem:

```mermaid
flowchart TD
    Start["Permission denied error"] --> Check["Check if subject has<br>necessary permissions"]
    Check -->|Use kubectl auth can-i| Result{"Has permissions?"}
    
    Result -->|No| Investigate["Investigate missing permissions"]
    Result -->|Yes| Other["Check other factors:<br>- PodSecurity<br>- Admission webhooks<br>- NetworkPolicies"]
    
    Investigate --> CheckRoles["Check Roles/ClusterRoles"]
    Investigate --> CheckBindings["Check RoleBindings/<br>ClusterRoleBindings"]
    
    CheckRoles --> AddRole["Add missing permissions<br>to Role/ClusterRole"]
    CheckBindings --> CreateBinding["Create missing binding<br>to connect subject to role"]
    
    AddRole --> Verify["Verify permissions<br>with kubectl auth can-i"]
    CreateBinding --> Verify
    
    Other --> CheckPolicy["Check policy engines<br>(Kyverno/OPA)"]
    Other --> CheckAdmission["Check admission controllers"]
    
    style Start fill:#f99,stroke:#333,stroke-width:2px
    style Result fill:#bbf,stroke:#333,stroke-width:2px
    style Verify fill:#9f9,stroke:#333,stroke-width:2px
```

## 🎓 Understanding the Relationship Between RBAC Components

```mermaid
erDiagram
    SUBJECT ||--o{ ROLEBINDING : "bound by"
    SUBJECT ||--o{ CLUSTERROLEBINDING : "bound by"
    ROLE ||--o{ ROLEBINDING : "referenced by"
    CLUSTERROLE ||--o{ ROLEBINDING : "referenced by"
    CLUSTERROLE ||--o{ CLUSTERROLEBINDING : "referenced by"
    
    SUBJECT {
        string name
        string kind
        string namespace
    }
    
    ROLE {
        string name
        string namespace
        array rules
    }
    
    CLUSTERROLE {
        string name
        array rules
        object aggregationRule
    }
    
    ROLEBINDING {
        string name
        string namespace
        object subjects
        object roleRef
    }
    
    CLUSTERROLEBINDING {
        string name
        object subjects
        object roleRef
    }
```