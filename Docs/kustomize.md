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

## Comprehensive Guide to Patching Strategies in Kustomize

Patching is at the heart of Kustomize's customization capabilities. There are two primary patching strategies that you need to understand to effectively use Kustomize: Strategic Merge Patching and JSON 6902 Patching. Each serves different purposes and has distinct advantages.

### Strategic Merge Patching

Strategic Merge Patches are partial Kubernetes resource definitions that merge with the original resources according to special semantics defined by Kubernetes. They're particularly powerful for working with complex Kubernetes resources.

#### Characteristics of Strategic Merge Patches:

1. **Structure**: Looks like a partial Kubernetes resource with the same apiVersion, kind, and metadata.name
2. **Merge Semantics**: Uses Kubernetes-aware merging that understands resource structures
3. **List Handling**: Special handling for lists with merge keys (like containers) versus primitive lists
4. **Field Removal**: Can delete fields using special directives

#### Implementation Options:

**1. Using Inline Patches in kustomization.yaml (Modern Method)**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: car-app
    spec:
      replicas: 3
      template:
        metadata:
          labels:
            env: development
        spec:
          containers:
          - name: car-app
            image: alvin254/car-app:v1.0.0-dev
  target:
    kind: Deployment
    name: car-app
```

**2. Using External Patch Files (Recommended for Complex Patches)**

First, create a patch file in your overlay directory:

```yaml
# ./kustomize/overlays/dev/patches/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-app
  namespace: frontend  # Must exactly match the original resource
spec:
  replicas: 1
  template:
    metadata:
      labels:
        env: development
    spec:
      containers:
      - name: car-app
        image: alvin254/car-app:v1.0.0-dev
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "250m"
```

Then reference this patch file in your kustomization.yaml:

```yaml
# ./kustomize/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patches:
- path: patches/deployment-patch.yaml
```

**3. Using the Legacy patchesStrategicMerge Syntax (Deprecated)**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patchesStrategicMerge:
- patches/deployment-patch.yaml
```

#### Special Merging Behaviors

Strategic merge patches have special behaviors worth understanding:

**1. List Merging Logic**

Lists in Kubernetes resources are handled differently based on whether they have a "merge key":

- **Lists with merge keys** (like containers, which use name as a merge key): Items in the list are merged based on the merge key
- **Lists without merge keys** (like args): The entire list is replaced

Example of updating a specific container in a pod:

```yaml
spec:
  template:
    spec:
      containers:
      - name: car-app  # Identifies which container to patch
        resources:     # These settings merge with existing container
          limits:
            memory: "1Gi"
```

**2. Field Deletion**

To remove a field or element, use the `$patch: delete` directive:

```yaml
spec:
  template:
    spec:
      containers:
      - name: car-app
        env:
        - name: DEBUG
          $patch: delete  # Removes this environment variable
```

**3. Replacing Entire Lists**

To replace an array instead of merging, use `$patch: replace`:

```yaml
spec:
  template:
    spec:
      containers:
      - name: car-app
        args:
          $patch: replace
          - "--new-arg1"
          - "--new-arg2"
```

#### Real-World Environment-Specific Patch Example

For different environments, you can create specialized patches:

**Dev Environment (./kustomize/overlays/dev/patches/deployment-patch.yaml):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-app
spec:
  replicas: 1  # Fewer replicas for dev
  template:
    metadata:
      labels:
        env: development
    spec:
      containers:
      - name: car-app
        image: alvin254/car-app:v1.0.0-dev  # Development image
        env:
        - name: DEBUG
          value: "true"  # Enable debugging
        resources:
          limits:
            memory: "512Mi"  # Lower resource limits
            cpu: "500m"
```

**Production Environment (./kustomize/overlays/prod/patches/deployment-patch.yaml):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-app
spec:
  replicas: 5  # More replicas for production
  template:
    metadata:
      labels:
        env: production
    spec:
      containers:
      - name: car-app
        image: alvin254/car-app:v1.0.0  # Stable image
        resources:
          limits:
            memory: "2Gi"  # Higher resource limits
            cpu: "2"
        readinessProbe:  # Enhanced health checks
          periodSeconds: 5  # More frequent checks
```

### JSON 6902 Patching

JSON 6902 patches (named after the RFC standard) provide a more surgical approach to modifying resources. They're especially useful when you need precise control over modifications, including working with arrays and complex nested structures.

#### Characteristics of JSON 6902 Patches:

1. **Operation-Based**: Uses explicit operations (add, remove, replace, move, copy, test)
2. **Path-Based**: Uses JSON paths to precisely target fields
3. **Array Indexing**: Can target specific array elements by index
4. **Independence**: Doesn't require knowledge of Kubernetes-specific merge semantics

#### Implementation Options:

**1. Using Inline JSON 6902 Patches**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patches:
- target:
    kind: Deployment
    name: car-app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 3
    - op: replace
      path: /spec/template/spec/containers/0/image
      value: alvin254/car-app:v1.0.0-dev
    - op: add
      path: /spec/template/spec/containers/0/env/-
      value:
        name: DEBUG
        value: "true"
```

**2. Using External JSON Patch Files**

First, create a JSON patch file:

```json
// ./kustomize/overlays/dev/patches/deployment-patch.json
[
  {
    "op": "replace",
    "path": "/spec/replicas",
    "value": 1
  },
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/image",
    "value": "alvin254/car-app:v1.0.0-dev"
  },
  {
    "op": "add",
    "path": "/spec/template/metadata/labels/env",
    "value": "development"
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/resources",
    "value": {
      "limits": {
        "memory": "512Mi",
        "cpu": "500m"
      },
      "requests": {
        "memory": "256Mi",
        "cpu": "250m"
      }
    }
  }
]
```

Then reference it in your kustomization:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patches:
- target:
    kind: Deployment
    name: car-app
  path: patches/deployment-patch.json
```

**3. Using the Legacy patchesJson6902 Syntax (Deprecated)**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: car-app
  path: patches/deployment-patch.json
```

#### JSON 6902 Operations

JSON 6902 patches support six operations:

1. **add**: Adds a new field or array element
    
    ```json
    {"op": "add", "path": "/spec/template/spec/containers/0/env/-", "value": {"name": "LOG_LEVEL", "value": "debug"}}
    ```
    
2. **remove**: Removes a field or array element
    
    ```json
    {"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}
    ```
    
3. **replace**: Updates an existing field
    
    ```json
    {"op": "replace", "path": "/spec/replicas", "value": 3}
    ```
    
4. **move**: Relocates a value from one path to another
    
    ```json
    {"op": "move", "path": "/spec/template/metadata/annotations/newKey", "from": "/spec/template/metadata/annotations/oldKey"}
    ```
    
5. **copy**: Copies a value from one path to another
    
    ```json
    {"op": "copy", "path": "/spec/template/metadata/labels/newLabel", "from": "/spec/template/metadata/labels/existingLabel"}
    ```
    
6. **test**: Validates that a value matches (useful for ensuring assumptions before applying patches)
    
    ```json
    {"op": "test", "path": "/spec/replicas", "value": 1}
    ```
    

#### JSON Path Syntax

Understanding JSON path syntax is crucial for effective patching:

- Paths start with `/` and sections are separated by `/`
- Array elements can be referenced by index: `/containers/0` (first element)
- The special token `-` refers to the end of an array: `/containers/-` (append)
- Special characters in field names need to be escaped with `~`:
    - `~0` for `~`
    - `~1` for `/`

#### Complex JSON Patch Examples

**Adding Probes:**

```json
[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/startupProbe",
    "value": {
      "httpGet": {
        "path": "/health",
        "port": 8080
      },
      "failureThreshold": 30,
      "periodSeconds": 10
    }
  }
]
```

**Working with Array Indices:**

```json
[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/ports/0/containerPort",
    "value": 8080
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/ports/-",
    "value": {
      "containerPort": 9090,
      "name": "metrics"
    }
  }
]
```

### Choosing Between Patching Strategies

Both patching strategies have their place in a Kustomize workflow. Here's when to use each:

**Use Strategic Merge Patches when:**

- You need to patch Kubernetes-native resources where merge semantics matter
- The changes are straightforward and follow the same structure as the original resource
- You're working primarily with adding or updating fields
- Your team is more familiar with YAML and Kubernetes resource formats

**Use JSON 6902 Patches when:**

- You need precise control over array operations (adding/removing specific items)
- Working with complex nested structures where paths are clearer than partial resources
- You need to perform operations like move, copy, or test
- The target resource has unusual merge semantics or isn't a native Kubernetes type
- You need to conditionally apply patches based on value tests

### Best Practices for Patching

1. **Organize Patches by Resource Type** Keep patches organized either by resource type or by logical change:
    
    ```
    overlays/dev/patches/
    ├── deployment-resources.yaml
    ├── deployment-image.yaml
    ├── deployment-env-vars.yaml
    └── service-type.yaml
    ```
    
2. **Use Meaningful Patch Filenames** Name patches clearly to indicate what they modify:
    
    ```
    car-app-replicas-patch.yaml
    car-app-resources-patch.yaml
    car-app-image-patch.json
    ```
    
3. **Choose the Right Strategy for the Job**
    
    - Strategic merge for broad changes that follow resource structure
    - JSON 6902 for precise, targeted changes, especially involving arrays
4. **Keep Patches Minimal and Focused** Each patch should have a single responsibility. This improves readability and reduces conflicts.
    
5. **Document Complex Patches** Add comments explaining non-obvious patch operations:
    
    ```yaml
    # This patch modifies the readiness probe timing for production environment
    # to ensure proper behavior during rolling updates
    ```
    
6. **Test Patches Individually** Before applying multiple patches, test each patch individually to verify its effect.
    
7. **Use YAML for Strategic Merge and JSON for JSON 6902** While both formats work for both patch types, the convention is to use:
    
    - `.yaml` for strategic merge patches (which are themselves YAML resources)
    - `.json` for JSON 6902 patches (which follow the JSON Patch spec)
8. **Watch for Path Accuracy** JSON paths and resource structures must exactly match the target resources, including case sensitivity.
    
9. **Consider Patch Order** Patches are applied in the order they're listed in the kustomization.yaml file. Order matters when patches build on each other.
    
10. **Namespace Consistency** When targeting namespaced resources, ensure the namespace in your patch target selector matches the actual namespace.
    

### Advanced Patching Techniques

#### 1. Multi-resource Targeting

Target multiple resources with a single patch:

```yaml
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: not-important
    spec:
      template:
        spec:
          containers:
          - name: not-important
            resources:
              limits:
                memory: "1Gi"
  target:
    kind: Deployment
    labelSelector: "app=car-app"
```

#### 2. Combining Patch Types

You can use both strategic merge and JSON 6902 patches in the same kustomization:

```yaml
patches:
# Strategic merge patch
- path: patches/deployment-resources.yaml
# JSON 6902 patch  
- target:
    kind: Deployment
    name: car-app
  path: patches/deployment-containers.json
```

#### 3. Environment Variable Management

For dev, staging, and production, manage environment variables differently:

**Dev (./kustomize/overlays/dev/patches/deployment-env.yaml):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-app
spec:
  template:
    spec:
      containers:
      - name: car-app
        env:
        - name: LOG_LEVEL
          value: "debug"
        - name: ENVIRONMENT
          value: "development"
        - name: FEATURE_FLAGS
          value: "new-ui=true,beta-api=true"
```

**Staging (./kustomize/overlays/staging/patches/deployment-env.yaml):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-app
spec:
  template:
    spec:
      containers:
      - name: car-app
        env:
        - name: LOG_LEVEL
          value: "info"
        - name: ENVIRONMENT
          value: "staging"
        - name: FEATURE_FLAGS
          value: "new-ui=true,beta-api=false"
```

**Production (./kustomize/overlays/prod/patches/deployment-env.yaml):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-app
spec:
  template:
    spec:
      containers:
      - name: car-app
        env:
        - name: LOG_LEVEL
          value: "warn"
        - name: ENVIRONMENT
          value: "production"
        - name: FEATURE_FLAGS
          value: "new-ui=false,beta-api=false"
```

#### 4. Selective Field Replacement

For more complex scenarios, JSON 6902 patches can selectively replace nested fields:

```json
[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/livenessProbe/initialDelaySeconds",
    "value": 30
  },
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/livenessProbe/periodSeconds",
    "value": 15
  }
]
```

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