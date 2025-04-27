# Kustomize: The Comprehensive Guide

## Introduction to Kustomize

Kustomize is a powerful, template-free configuration management tool for Kubernetes manifests. Unlike template-based tools that mix logic and structure, Kustomize embraces a purely declarative approach. It treats Kubernetes YAML as data that can be transformed through a series of operations, preserving the original structure while enabling customizations for different environments.

## Kustomize Architecture Overview

```mermaid
graph TD
    A[Base Resources] --> B[Kustomization Engine]
    C[Transformers] --> B
    D[Generators] --> B
    B --> E[Final Manifests]
    
    subgraph "Inputs"
    A
    C
    D
    end
    
    subgraph "Processing"
    B
    end
    
    subgraph "Output"
    E
    end
```

## Core Concepts and Components

### 1. Base and Overlays Pattern

The foundational design pattern in Kustomize is the base/overlays structure:

- **Base**: Contains the common, foundational resources shared across all environments
- **Overlays**: Environment-specific directories that customize the base 

```mermaid
graph TD
    B[Base] --> |Referenced by| D[Dev Overlay]
    B --> |Referenced by| S[Staging Overlay]
    B --> |Referenced by| P[Production Overlay]
    
    D --> |kubectl apply -k| DK[Kubernetes Dev]
    S --> |kubectl apply -k| SK[Kubernetes Staging]
    P --> |kubectl apply -k| PK[Kubernetes Production]
    
    style B fill:#d0e0ff,stroke:#0066cc
    style D fill:#d0ffe0,stroke:#00cc66
    style S fill:#ffe0d0,stroke:#cc6600
    style P fill:#ffd0d0,stroke:#cc0000
```

A typical directory structure implementing this pattern looks like:

```
./kustomize/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

### 2. The kustomization.yaml File

The `kustomization.yaml` file is the configuration that controls how Kustomize processes resources:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Resources to include
resources:
- deployment.yaml
- service.yaml

# Common labels
labels:
  pairs:
    app: my-application
    environment: production

# Add name prefixes
namePrefix: prod-

# Patches to apply
patches:
- path: increase-replicas.yaml
  target:
    kind: Deployment
    name: my-deployment
```

### 3. Resource Processing Flow

The following diagram shows how Kustomize processes resources:

```mermaid
flowchart TD
    A[Load Resources] --> B[Apply NamePrefix/Suffix]
    B --> C[Apply Labels/Annotations]
    C --> D[Generate ConfigMaps/Secrets]
    D --> E[Apply Patches]
    E --> F[Process Replacements]
    F --> G[Apply Image Transformations]
    G --> H[Adjust Replicas]
    H --> I[Add Hash Suffixes]
    I --> J[Output Final YAML]
```

## Kustomize Path Security Model

### Why Path Restrictions Exist

You asked specifically why you can't reference files outside your kustomization directory. This is due to Kustomize's path security model, which is a deliberate design choice.

```mermaid
graph TD
    subgraph "Kustomize Security Boundary" 
    A[kustomization.yaml]
    B[Allowed: Resources in same directory]
    C[Allowed: Resources in subdirectories]
    end
    
    D[Blocked: Resources outside directory]
    E[Blocked: Resources in parent directories]
    
    A --> B
    A --> C
    A -. ❌ Access Denied .-> D
    A -. ❌ Access Denied .-> E
    
    style A fill:#f9f9f9,stroke:#333333
    style B fill:#d0ffe0,stroke:#00cc66
    style C fill:#d0ffe0,stroke:#00cc66
    style D fill:#ffd0d0,stroke:#cc0000
    style E fill:#ffd0d0,stroke:#cc0000
```

When you try to reference external resources like this:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../pkg/pkg1/manifests/deployment.yaml
- ../../pkg/pkg1/manifests/service-account.yaml
```

Kustomize will produce an error like:

```
error: accumulating resources: accumulation err='accumulating resources from 
'../../pkg/pkg1/manifests/deployment.yaml': security; file not in or below current directory'
```

### Reasons Behind This Security Model

This restriction serves critical purposes:

1. **Security Against Path Traversal Attacks**: Prevents malicious kustomization files from accessing unauthorized files outside their boundary.

2. **Hermetic Builds**: Ensures reproducibility by making builds self-contained and independent of arbitrary directory structures.

3. **Clear Resource Boundaries**: Creates explicit boundaries around what files can affect your build.

4. **Modularity**: Encourages proper self-contained modules that can be shared and reused.

## Solving the Path Restriction Issue

To address the path restriction issue, you have several options:

### Option 1: Copy Resources to Your Base Directory

```mermaid
graph TD
    A[Original Resources in pkg/] --> |cp command| B[Copied Resources in kustomize/base/]
    B --> C[kustomization.yaml references local copies]
    C --> D[Kustomize can now process resources]
    
    style A fill:#ffeecc,stroke:#ff9900
    style B fill:#ccffcc,stroke:#00cc66
    style C fill:#cceeff,stroke:#0099ff
    style D fill:#d0e0ff,stroke:#0066cc
```

Implementation steps:

```bash
mkdir -p kustomize/base/manifests
cp pkg/pkg1/manifests/deployment.yaml kustomize/base/manifests/
cp pkg/pkg1/manifests/service-account.yaml kustomize/base/manifests/
# Copy remaining files...
```

Then update your `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- manifests/deployment.yaml
- manifests/service-account.yaml
# Additional resources...
```

### Option 2: Restructure Your Repository

```mermaid
graph TD
    A[Original Structure] --> B[Restructured Repository]
    
    subgraph "Original Structure"
    A1[kustomize/base]
    A2[pkg/pkg1/manifests]
    A3[pkg/pkg2/manifests]
    end
    
    subgraph "Restructured Repository"
    B1[bases/pkg1/]
    B2[bases/pkg2/]
    B3[overlays/dev]
    B4[overlays/prod]
    end
    
    B3 --> |references| B1
    B3 --> |references| B2
    B4 --> |references| B1
    B4 --> |references| B2
    
    style A1 fill:#ffcccc,stroke:#ff6666
    style A2 fill:#ffcccc,stroke:#ff6666
    style A3 fill:#ffcccc,stroke:#ff6666
    
    style B1 fill:#ccffcc,stroke:#00cc66
    style B2 fill:#ccffcc,stroke:#00cc66
    style B3 fill:#cceeff,stroke:#0099ff
    style B4 fill:#cceeff,stroke:#0099ff
```

### Option 3: Run Kustomize from a Higher Directory

```mermaid
flowchart TD
    A[project root directory] --> |cd to root| B[Run kubectl kustomize ./kustomize/base]
    B --> C[Security boundary now includes pkg/]
    C --> D[References to ../../pkg/ now work]
    
    style A fill:#ffffcc,stroke:#cccc00
    style B fill:#d0e0ff,stroke:#0066cc
    style C fill:#ccffcc,stroke:#00cc66
    style D fill:#ccffcc,stroke:#00cc66
```

However, this approach has drawbacks:
- Depends on specific directory structure
- Reduces portability
- Works around security rather than working with it

## The Complete Kustomize Workflow

### 1. Building and Rendering

```mermaid
flowchart LR
    A[kustomization.yaml] --> B[kustomize build]
    B --> C[Generated YAML]
    C --> D[kubectl apply]
    D --> E[Kubernetes Cluster]
    
    style A fill:#d0e0ff,stroke:#0066cc
    style B fill:#ffd0e0,stroke:#cc0066
    style C fill:#d0ffe0,stroke:#00cc66
    style D fill:#ffe0d0,stroke:#cc6600
    style E fill:#e0d0ff,stroke:#6600cc
```

### 2. Overlay Inheritance and Composition

```mermaid
graph TD
    B[Base kustomization.yaml] --> D[Dev Overlay]
    B --> P[Production Overlay]
    
    subgraph "Base - kustomize/base"
    B1[deployment.yaml]
    B2[service.yaml]
    B3[configmap.yaml]
    B --> B1
    B --> B2
    B --> B3
    end
    
    subgraph "Dev Overlay - kustomize/overlays/dev"
    D1[dev-specific-config.yaml]
    D2[lower-replicas.yaml]
    D --> D1
    D --> D2
    end
    
    subgraph "Production Overlay - kustomize/overlays/prod"
    P1[prod-specific-config.yaml]
    P2[higher-replicas.yaml]
    P3[resource-limits.yaml]
    P --> P1
    P --> P2
    P --> P3
    end
    
    style B fill:#d0e0ff,stroke:#0066cc
    style D fill:#d0ffe0,stroke:#00cc66
    style P fill:#ffd0d0,stroke:#cc0000
```

## Advanced Kustomize Features

### 1. Patches

Kustomize supports multiple patch types:

```mermaid
graph TD
    A[Patches] --> B[Strategic Merge Patches]
    A --> C[JSON 6902 Patches]
    
    subgraph "Strategic Merge Patch"
    B1[Partial YAML that merges with target]
    B --> B1
    end
    
    subgraph "JSON 6902 Patch"
    C1[Operation-based JSON patch format]
    C2[Precise field targeting]
    C --> C1
    C --> C2
    end
    
    style A fill:#d0e0ff,stroke:#0066cc
    style B fill:#ffe0d0,stroke:#cc6600
    style C fill:#d0ffe0,stroke:#00cc66
```

Example of both patch types:

```yaml
# Strategic Merge Patch
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: deployment
    spec:
      replicas: 3
  target:
    kind: Deployment
    name: my-deployment

# JSON 6902 Patch  
patches:
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
  target:
    kind: Deployment
    name: another-deployment
```

### 2. ConfigMap and Secret Generation

```mermaid
graph TD
    A[ConfigMap and Secret Generators] --> B[Input Sources]
    B --> C[Files]
    B --> D[Literals]
    B --> E[Environment Files]
    
    A --> F[Output]
    F --> G[Unique resources with hash suffix]
    F --> H[Auto-updated when inputs change]
    
    style A fill:#d0e0ff,stroke:#0066cc
    style B fill:#ffe0d0,stroke:#cc6600
    style F fill:#d0ffe0,stroke:#00cc66
```

Example generator configuration:

```yaml
configMapGenerator:
- name: app-config
  files:
  - config.properties
  literals:
  - API_URL=https://api.example.com
  - DEBUG=true

secretGenerator:
- name: app-secrets
  files:
  - tls.key
  - tls.crt
  envs:
  - .env.secret
```

### 3. Field Replacements

Modern Kustomize uses replacements for variable substitution:

```mermaid
flowchart LR
    A[Source Field] --> |Value extracted| B[Replacement Engine]
    B --> |Value inserted| C[Target Field]
    
    style A fill:#ffe0d0,stroke:#cc6600
    style B fill:#d0e0ff,stroke:#0066cc
    style C fill:#d0ffe0,stroke:#00cc66
```

```yaml
replacements:
- source:
    kind: ConfigMap
    name: app-config
    fieldPath: data.log-level
  targets:
  - select:
      kind: Deployment
    fieldPaths:
    - spec.template.spec.containers.[name=logger].env.[name=LOG_LEVEL].value
```

## Implementation for Your Repository Structure

### Original Problem Structure

```mermaid
graph TD
    subgraph "Repository Structure"
    A[kustomize/base/kustomization.yaml]
    B[pkg/pkg1/manifests/deployment.yaml]
    C[pkg/pkg1/manifests/service-account.yaml]
    D[pkg/pkg2/manifests/deployment.yaml]
    end
    
    A -. ❌ Cannot reference outside base .-> B
    A -. ❌ Cannot reference outside base .-> C
    A -. ❌ Cannot reference outside base .-> D
    
    style A fill:#ffd0d0,stroke:#cc0000
    style B fill:#d0ffe0,stroke:#00cc66
    style C fill:#d0ffe0,stroke:#00cc66
    style D fill:#d0ffe0,stroke:#00cc66
```

### Solution Structure

```mermaid
graph TD
    subgraph "Recommended Solution"
    A[kustomize/base/kustomization.yaml]
    B[kustomize/base/manifests/pkg1/deployment.yaml]
    C[kustomize/base/manifests/pkg1/service-account.yaml]
    D[kustomize/base/manifests/pkg2/deployment.yaml]
    end
    
    A --> B
    A --> C
    A --> D
    
    E[Copy Files] --> |Creates| B
    E --> |Creates| C
    E --> |Creates| D
    
    style A fill:#d0ffe0,stroke:#00cc66
    style B fill:#d0ffe0,stroke:#00cc66
    style C fill:#d0ffe0,stroke:#00cc66
    style D fill:#d0ffe0,stroke:#00cc66
    style E fill:#d0e0ff,stroke:#0066cc
```

Implementation steps:

```bash
# Create directory structure
mkdir -p kustomize/base/manifests/pkg1
mkdir -p kustomize/base/manifests/pkg2

# Copy files
cp pkg/pkg1/manifests/*.yaml kustomize/base/manifests/pkg1/
cp pkg/pkg2/manifests/*.yaml kustomize/base/manifests/pkg2/

# Update kustomization.yaml
```

Updated `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- manifests/pkg1/deployment.yaml
- manifests/pkg1/service-account.yaml
- manifests/pkg1/role.yaml
- manifests/pkg1/role-binding.yaml
- manifests/pkg2/deployment.yaml
- manifests/pkg2/car-service.yaml

labels:
  pairs:
    app: car-app
    environment: base
```

## Kustomize Under the Hood

### Resource Loading and Processing

```mermaid
flowchart TB
    A[Load kustomization.yaml] --> B[Resolve resources]
    B --> C[Apply transformers]
    C --> D[Validate resources]
    D --> E[Generate output]
    
    subgraph "Resource Resolution"
    B1[Load local files]
    B2[Process remote resources]
    B3[Build bases/components]
    B4[Check security constraints]
    B --> B1
    B --> B2
    B --> B3
    B --> B4
    end
    
    subgraph "Transformer Pipeline"
    C1[Name transformers]
    C2[Label transformers]
    C3[Image transformers]
    C4[Patch transformers]
    C5[Replacement transformers]
    C --> C1
    C --> C2
    C --> C3
    C --> C4
    C --> C5
    end
    
    style A fill:#ffe0d0,stroke:#cc6600
    style B fill:#d0e0ff,stroke:#0066cc
    style C fill:#d0ffe0,stroke:#00cc66
    style D fill:#e0d0ff,stroke:#6600cc
    style E fill:#ffd0d0,stroke:#cc0000
```

### Transformer Order

The order of transformers is carefully designed to handle dependencies properly:

1. Namespace transformer
2. Name transformers (prefix/suffix)
3. Label and annotation transformers
4. Generator transformers (ConfigMaps/Secrets)
5. Patch transformers (strategic merge/JSON)
6. Replacement/variable transformers
7. Image transformers
8. Replica transformers
9. Hash transformers

## Best Practices for Kustomize

### Directory Structure Best Practices

```mermaid
graph TD
    A[Project Root] --> B[bases/]
    A --> C[components/]
    A --> D[overlays/]
    
    B --> B1[app-base/]
    B --> B2[db-base/]
    
    C --> C1[monitoring/]
    C --> C2[logging/]
    C --> C3[security/]
    
    D --> D1[dev/]
    D --> D2[staging/]
    D --> D3[prod/]
    
    D3 --> D3A[us-east/]
    D3 --> D3B[us-west/]
    
    style A fill:#f9f9f9,stroke:#333333
    style B fill:#d0e0ff,stroke:#0066cc
    style C fill:#e0d0ff,stroke:#6600cc
    style D fill:#ffe0d0,stroke:#cc6600
```

### Base Design Principles

- Keep bases minimal and focused on common elements
- Include only essential resources in the base
- Avoid environment-specific configurations in the base
- Structure bases around logical components or applications

### Overlay Organization

- Create separate overlays for each environment or variant
- Keep overlay-specific patches in the overlay directory
- Use clear, consistent naming conventions
- Minimize the number of patches per overlay

### Path Management 

- Use relative paths within the kustomization directory
- Keep resources close to the kustomization that uses them
- Avoid complex directory traversal
- Remember the security boundary: all references must be within or below the kustomization directory

## Troubleshooting Kustomize Issues

### Common Problems and Solutions

```mermaid
flowchart TD
    A[Common Kustomize Issues] --> B[Path Resolution]
    A --> C[Patch Application]
    A --> D[Duplicate Resources]
    A --> E[Deprecated Fields]
    
    B --> B1[Ensure resources within boundary]
    B --> B2[Verify paths relative to kustomization.yaml]
    B --> B3[Run from correct directory]
    
    C --> C1[Verify target identifiers]
    C --> C2[Check patch format]
    C --> C3[Ensure metadata.name matches]
    
    D --> D1[Use namePrefixes]
    D --> D2[Use different namespaces]
    D --> D3[Check for duplicate references]
    
    E --> E1[Run kustomize edit fix]
    E --> E2[Check version compatibility]
    E --> E3[Review migration guides]
```

### Debugging Techniques

- Use `kubectl kustomize . | less` to inspect output
- Add `--enable-alpha-plugins --load-restrictor LoadRestrictionsNone` for advanced debugging
- Count resources with `kubectl kustomize . | grep kind: | sort | uniq -c`
- Validate with `kubectl kustomize . | kubectl apply --dry-run=client -f -`

## Working with the Command Line

### Essential Commands

```mermaid
graph LR
    A[Kustomize Commands] --> B[kustomize build]
    A --> C[kubectl kustomize]
    A --> D[kubectl apply -k]
    A --> E[kustomize edit]
    
    B --> B1[Generate YAML]
    C --> C1[View manifests]
    D --> D1[Apply to cluster]
    E --> E1[Modify kustomization]
    
    E --> E2[add resource]
    E --> E3[set nameprefix]
    E --> E4[fix]
    
    style A fill:#d0e0ff,stroke:#0066cc
    style B fill:#ffe0d0,stroke:#cc6600
    style C fill:#d0ffe0,stroke:#00cc66
    style D fill:#e0d0ff,stroke:#6600cc
    style E fill:#ffd0d0,stroke:#cc0000
```

Common commands:

```bash
# Build and view manifests
kubectl kustomize ./kustomize/overlays/dev

# Apply to cluster
kubectl apply -k ./kustomize/overlays/prod

# Edit kustomization file
kustomize edit add resource deployment.yaml
kustomize edit set nameprefix prod-
kustomize edit fix
```

## Migrating from Legacy Fields

Kustomize evolves over time, and some fields are deprecated:

| Deprecated Field | Modern Replacement | Description |
|------------------|-------------|-------------|
| `commonLabels` | `labels` | Labels for all resources |
| `commonAnnotations` | `annotations` | Annotations for all resources |
| `patchesStrategicMerge` | `patches` | Strategic merge patches |
| `patchesJson6902` | `patches` | JSON patches |
| `vars` | `replacements` | Variable substitution |

The `kustomize edit fix` command helps migrate to newer syntax.

## Real-World Examples

### Multi-Tier Application Example

```mermaid
graph TD
    subgraph "Base Structure"
    B[base kustomization.yaml]
    B --> BF[frontend deployment]
    B --> BS[frontend service]
    B --> BB[backend deployment]
    B --> BBS[backend service]
    B --> BD[database statefulset]
    B --> BDS[database service]
    end
    
    subgraph "Development Overlay"
    D[dev kustomization.yaml]
    D --> DF[local database config]
    D --> DL[debug logging]
    D --> DR[lower replicas]
    end
    
    subgraph "Production Overlay"
    P[prod kustomization.yaml]
    P --> PD[high availability DB]
    P --> PR[higher replicas]
    P --> PM[monitoring sidecars]
    P --> PC[CDN configuration]
    end
    
    B --> D
    B --> P
    
    style B fill:#d0e0ff,stroke:#0066cc
    style D fill:#d0ffe0,stroke:#00cc66
    style P fill:#ffd0d0,stroke:#cc0000
```

### Using Components for Cross-Cutting Concerns

```mermaid
graph TD
    B[Base Application] --> |Referenced by| D[Dev Overlay]
    B --> |Referenced by| P[Prod Overlay]
    
    M[Monitoring Component] --> |Applied to| D
    M --> |Applied to| P
    
    S[Security Component] --> |Applied to| P
    
    L[Logging Component] --> |Applied to| D
    L --> |Applied to| P
    
    style B fill:#d0e0ff,stroke:#0066cc
    style M fill:#ffe0d0,stroke:#cc6600
    style S fill:#e0d0ff,stroke:#6600cc
    style L fill:#ffd0ff,stroke:#cc00cc
    style D fill:#d0ffe0,stroke:#00cc66
    style P fill:#ffd0d0,stroke:#cc0000
```

## Conclusion

Kustomize's path security model, which prevents referencing files outside your kustomization directory, is a deliberate design feature that promotes security, reproducibility, and proper modularization. While it may initially seem restrictive, embracing this model leads to more robust, portable, and maintainable Kubernetes configurations.

The need to copy files into your base directory isn't a limitation but an invitation to structure your configuration for better modularity and security. By creating proper self-contained bases and leveraging overlays for customization, you can build a powerful, declarative configuration system that scales across environments while maintaining a single source of truth.

Whether you choose to copy files to maintain your current structure or embark on a more comprehensive restructuring to align with Kustomize's design philosophy, understanding the rationale behind the path restrictions empowers you to work effectively with the tool rather than fighting against its fundamental design principles.