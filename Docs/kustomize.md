# Comprehensive Guide to Kustomize: How It Works and Why Path Restrictions Matter

## Introduction to Kustomize

Kustomize is a standalone tool that lets you customize Kubernetes manifests without templates. It follows a declarative approach where you define the desired state using a `kustomization.yaml` file that specifies a series of transformations to apply to raw Kubernetes manifests. This approach preserves the original YAML files while allowing customizations for different environments.

## Core Architecture and Concepts

### Base and Overlays

Kustomize's fundamental architecture revolves around two key concepts:

1. **Base**: A directory containing a `kustomization.yaml` file and a set of basic Kubernetes resource files that serve as the common foundation.
    
2. **Overlays**: Directories containing `kustomization.yaml` files that reference and customize the base for specific environments (dev, staging, prod, etc.).
    

Your repository illustrates this structure:

```
./kustomize/
├── base/
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

This structure enables a single source of truth (the base) with environment-specific customizations (the overlays).

### The kustomization.yaml File

The `kustomization.yaml` file serves as the configuration that controls how Kustomize transforms resources. It can include:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Resources to be customized
resources:
- deployment.yaml
- service.yaml

# Add common labels to all resources
labels:
  pairs:
    app: my-application
    environment: production

# Add name prefixes
namePrefix: prod-

# Transformations to apply
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
```

## Kustomize Path Security Model - Addressing Your Question

### Why You Can't Reference External Files

You asked why you need to copy files into the base directory instead of referencing them directly like this:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../pkg/pkg1/manifests/deployment.yaml
- ../../pkg/pkg1/manifests/service-account.yaml
- ../../pkg/pkg1/manifests/role.yaml
- ../../pkg/pkg1/manifests/role-binding.yaml
- ../../pkg/pkg2/manifests/deployment.yaml
```

When you run Kustomize in your base directory, it enforces a fundamental security restriction: **all referenced files must be within or below the directory containing the kustomization.yaml file**. This is why attempting to access `../../pkg/pkg1/manifests/deployment.yaml` produces an error message like:

```
error: accumulating resources: accumulation err='accumulating resources from '../../pkg/pkg1/manifests/deployment.yaml': 
security; file '/home/alvo/Desktop/projects/cka-study/pkg/pkg1/manifests/deployment.yaml' is not in or below 
'/home/alvo/Desktop/projects/cka-study/kustomize/base'': 
must build at directory: '/home/alvo/Desktop/projects/cka-study/pkg/pkg1/manifests/deployment.yaml': file is not directory
```

### Reasons Behind This Restriction

This restriction is a deliberate design choice in Kustomize, not an accidental limitation. It serves several critical purposes:

1. **Security Against Path Traversal Attacks**: Path traversal is a common attack vector where manipulated file paths access unauthorized files. By restricting references to the current directory tree, Kustomize prevents these attacks, particularly important in automated CI/CD environments.
    
2. **Hermetic Builds and Portability**: Kustomize prioritizes reproducibility and portability. A kustomization directory should work consistently if copied elsewhere without depending on arbitrary directory structures above it. External dependencies break this portability model.
    
3. **Clear Resource Boundaries**: The restriction creates an explicit boundary around what files can affect your build. Without this, changes to arbitrary locations could have unexpected effects on your manifests.
    
4. **Intentional Modularity**: Kustomize encourages self-contained modules that can be versioned, shared, and reused. External references undermine this modularity.
    

### Solutions to the Path Restriction Issue

Given Kustomize's path security model, you have several options:

#### 1. Copy Resources to Your Base Directory (Recommended Approach)

```bash
mkdir -p kustomize/base/manifests
cp pkg/pkg1/manifests/deployment.yaml kustomize/base/manifests/
cp pkg/pkg1/manifests/service-account.yaml kustomize/base/manifests/
# Copy all needed resources...
```

Then reference these local copies:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- manifests/deployment.yaml
- manifests/service-account.yaml
# etc.
```

This approach:

- Creates a true "base" that's self-contained
- Makes your kustomization portable
- Provides explicit control over which version of resources you're using
- Enables you to modify the base copies without affecting the originals

#### 2. Run Kustomize from a Higher Directory

You could run Kustomize from a higher directory where both your kustomize directory and pkg directory are visible:

```bash
cd /home/alvo/Desktop/projects/cka-study
kubectl kustomize ./kustomize/base
```

However, this approach:

- Creates an implicit dependency on a specific directory structure
- Reduces portability
- Works around the security model rather than working with it
- Is generally considered a less robust practice

#### 3. Restructure Your Repository

Another option is to restructure your repository to align with Kustomize's model:

```
./
├── components/          # Reusable components
├── bases/              # Base configurations
│   ├── app-base/       # Your core application base
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── db-base/        # Database base
└── overlays/           # Environment-specific customizations
    ├── dev/
    ├── staging/
    └── prod/
```

#### 4. Use Git Submodules or Remote References

For truly shared resources used across multiple projects:

```yaml
# With Git submodules
resources:
- submodules/common-resources/deployment.yaml

# Or with remote references (newer Kustomize versions)
resources:
- https://github.com/org/repo/raw/main/manifests/deployment.yaml
```

## How Kustomize Works: The Complete Process

Now that we've addressed your specific question, let's explore the complete workflow of how Kustomize operates:

### 1. Resource Loading

When you run `kustomize build` or `kubectl kustomize`, Kustomize first:

1. Reads your `kustomization.yaml` file
2. Loads all resources specified in the `resources` field
3. Recursively processes any bases or components referenced
4. Validates that all file references comply with the path security model

### 2. Transformer Pipeline

After loading resources, Kustomize applies a series of transformers in a specific order:

1. **Namespace Transformer**: Adds or changes namespaces
2. **Name Transformers**: Applies prefixes/suffixes to resource names
3. **Label and Annotation Transformers**: Adds labels and annotations
4. **Patch Transformers**: Applies strategic merge patches and JSON patches
5. **Image Transformers**: Updates container image references
6. **Replica Transformers**: Modifies deployment replicas
7. **Configuration Generators**: Creates ConfigMaps and Secrets
8. **Value Addition Transformers**: Adds values like service selectors
9. **Hash Transformers**: Adds hash suffixes to ConfigMaps/Secrets
10. **Validation**: Performs final validation of resources

This pipeline creates a deterministic transformation process where each step builds on the previous one.

### 3. Patch Types and Application

Kustomize supports multiple patch types:

#### Strategic Merge Patches

These patches merge with the target using Kubernetes' strategic merge patch algorithm:

```yaml
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
```

#### JSON 6902 Patches

These patches apply RFC 6902 JSON Patch operations:

```yaml
patches:
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
  target:
    kind: Deployment
    name: my-deployment
```

### 4. ConfigMap and Secret Generation

Kustomize can generate ConfigMaps and Secrets from files, literals, or environment files:

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

These generators create unique resources with content hashes to ensure updates trigger redeployments.

### 5. Variable Substitution

Kustomize supports variable substitution with its `vars` field (deprecated) or more modern `replacements`:

```yaml
replacements:
- source:
    kind: Service
    name: my-service
    fieldPath: metadata.name
  targets:
  - select:
      kind: Deployment
    fieldPaths:
    - spec.template.spec.containers.[name=app].env.[name=SERVICE_NAME].value
```

This allows referencing values from one resource in another resource.

## Advanced Kustomize Features

### Components

Components are reusable collections of kustomizations that can be applied across multiple bases:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

components:
- ../components/monitoring
- ../components/istio
```

Components work similarly to bases but are designed for cross-cutting concerns rather than foundational resources.

### Field Replacements

Field replacements provide powerful ways to reference values across resources:

```yaml
replacements:
- source:
    kind: ConfigMap
    name: app-config
    version: v1
    fieldPath: data.log-level
  targets:
  - select:
      kind: Deployment
    fieldPaths:
    - spec.template.spec.containers.[name=logger].env.[name=LOG_LEVEL].value
```

### Inline Patches

Inline patches allow modifying resources without separate patch files:

```yaml
patches:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 3
```

### Multi-base Composition

Complex applications can combine multiple bases:

```yaml
resources:
- ../bases/frontend
- ../bases/backend
- ../bases/database
```

This allows modular composition of applications.

## CLI and Integration Options

### Kustomize CLI Commands

Kustomize provides several key CLI commands:

```bash
# Build and view manifests
kustomize build path/to/kustomization
kubectl kustomize path/to/kustomization

# Apply manifests
kubectl apply -k path/to/kustomization

# Edit kustomization files
kustomize edit add resource deployment.yaml
kustomize edit set nameprefix prod-
kustomize edit fix  # Update deprecated fields
```

### Integration with kubectl

Kubectl integrates Kustomize with the `-k` flag:

```bash
# View resources
kubectl get -k path/to/kustomization

# Apply resources
kubectl apply -k path/to/kustomization

# Delete resources
kubectl delete -k path/to/kustomization
```

## Deprecated Fields and Syntax Updates

Kustomize evolves over time, and some fields are deprecated in favor of new ones:

|Deprecated Field|Replacement|Description|
|---|---|---|
|`commonLabels`|`labels`|Adds labels to all resources|
|`commonAnnotations`|`annotations`|Adds annotations to all resources|
|`patchesStrategicMerge`|`patches`|Strategic merge patches|
|`patchesJson6902`|`patches`|JSON patches with extended syntax|
|`vars`|`replacements`|Variable substitution|

The `kustomize edit fix` command updates your kustomization files to use the newer syntax.

## Best Practices for Kustomize

### 1. Repository Structure

Design your repository with Kustomize's path restrictions in mind:

```
./
├── base/                # Common base resources
│   ├── manifests/       # All base manifests
│   └── kustomization.yaml
├── components/          # Reusable transformations
│   ├── monitoring/
│   ├── istio/
│   └── logging/
└── overlays/            # Environment-specific customizations
    ├── dev/
    ├── staging/
    └── prod/
```

### 2. Base Design Principles

- Keep bases minimal and focused on common elements
- Include only essential resources in the base
- Avoid environment-specific configurations in the base
- Structure bases around logical components or applications

### 3. Overlay Organization

- Create separate overlays for each environment or variant
- Keep overlay-specific patches in the overlay directory
- Use clear, consistent naming conventions
- Minimize the number of patches per overlay

### 4. Path Management

- Use relative paths within the kustomization directory
- Keep resources close to the kustomization that uses them
- Avoid complex directory traversal
- Remember the security boundary: all references must be within or below the kustomization directory

### 5. Version Control

- Commit all kustomization files and resources to version control
- Track base and overlay changes together
- Consider tagging stable versions of bases
- Use branch strategies aligned with your deployment environment separation

## Real-World Implementation Example

Let's apply this knowledge to your specific repository structure:

### Current Structure (With Path Issues)

```
./
├── kustomize/
│   ├── base/
│   │   └── kustomization.yaml  # References files outside its tree
│   └── overlays/
│       ├── dev/
│       ├── staging/
│       └── prod/
└── pkg/
    ├── pkg1/manifests/
    │   ├── deployment.yaml
    │   └── service-account.yaml
    └── pkg2/manifests/
        ├── deployment.yaml
        └── car-service.yaml
```

### Recommended Restructuring

#### Option 1: Copy Files (Minimal Change)

```
./
├── kustomize/
│   ├── base/
│   │   ├── manifests/
│   │   │   ├── deployment-pkg1.yaml  # Copied from pkg/pkg1/manifests/
│   │   │   ├── service-account.yaml  # Copied from pkg/pkg1/manifests/
│   │   │   └── deployment-pkg2.yaml  # Copied from pkg/pkg2/manifests/
│   │   └── kustomization.yaml        # References local files
│   └── overlays/
└── pkg/  # Original files remain unchanged
```

#### Option 2: Major Restructuring (Better Alignment)

```
./
├── bases/
│   ├── pkg1/
│   │   ├── manifests/
│   │   │   ├── deployment.yaml
│   │   │   └── service-account.yaml
│   │   └── kustomization.yaml
│   └── pkg2/
│       ├── manifests/
│       │   ├── deployment.yaml
│       │   └── car-service.yaml
│       └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml  # References ../../bases/pkg1 and ../../bases/pkg2
    ├── staging/
    └── prod/
```

### Implementation Plan

1. **Immediate Solution**: Copy required resources to `kustomize/base/manifests/` to make your current setup work:

```bash
mkdir -p kustomize/base/manifests
cp pkg/pkg1/manifests/*.yaml kustomize/base/manifests/pkg1/
cp pkg/pkg2/manifests/*.yaml kustomize/base/manifests/pkg2/
```

Then update your `kustomization.yaml`:

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
```

2. **Long-term Solution**: Gradually migrate to a structure that better aligns with Kustomize's model, separating bases by application or component rather than by package.

## Troubleshooting Common Kustomize Issues

### Path Resolution Issues

If you encounter errors like:

```
error: accumulating resources: accumulation err='accumulating resources from '../../pkg/pkg1/manifests/deployment.yaml': security; file not in or below current directory'
```

Check that:

- All referenced files are within or below the kustomization directory
- Paths are relative to the kustomization.yaml location
- You're running the kustomize command from the correct directory

### Patch Application Failures

If patches aren't applying correctly:

- Verify target resource identifiers (group, version, kind, name) match exactly
- Check that patches are properly formatted for their type
- For strategic merge patches, ensure the metadata.name matches the target
- For JSON patches, validate the JSON path expressions

### Multiple Resources with Same ID

If you see errors about duplicate resources:

- Add namePrefixes or nameSuffixes to distinguish resources
- Use different namespaces for duplicate resources
- Check if you're referencing the same resource multiple times

### Deprecated Field Warnings

If you receive warnings about deprecated fields:

- Run `kustomize edit fix` to update to newer syntax
- Review the Kustomize version compatibility
- Check release notes for migration guidance

## Conclusion

Kustomize's path security model, which prevents referencing files outside your kustomization directory, is a deliberate feature that promotes security, reproducibility, and proper modularization. While it may initially seem restrictive, particularly when transitioning from other directory structures, embracing this model leads to more robust, portable, and maintainable Kubernetes configurations.

The need to copy files into your base directory isn't a limitation but an invitation to structure your configuration for better modularity and security. By creating proper self-contained bases and leveraging overlays for customization, you can build a powerful, declarative configuration system that scales across environments while maintaining a single source of truth.

Whether you choose to copy files to maintain your current structure or embark on a more comprehensive restructuring to align with Kustomize's design philosophy, understanding the rationale behind the path restrictions empowers you to work effectively with the tool rather than fighting against its fundamental design principles.