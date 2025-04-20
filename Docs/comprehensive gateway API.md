
# Comprehensive Guide to Kubernetes Gateway API

## Introduction

The Gateway API represents the evolution of Kubernetes networking, providing a more expressive, extensible, and role-oriented approach to managing traffic compared to traditional Ingress resources. It introduces a sophisticated model for routing external traffic to Kubernetes services, offering fine-grained control through a hierarchy of purpose-built resources.

This document provides a thorough exploration of the Gateway API, from fundamental concepts to advanced configurations, with practical examples that demonstrate its power and flexibility in modern cloud-native environments.

## Table of Contents

1. Gateway API Fundamentals
2. Core Resources and Architecture
3. Basic Setup and Configuration
4. TLS Termination and Certificate Management
5. HTTP Routing Essentials
6. Header-Based Routing
7. Path-Based Routing Techniques
8. Traffic Weighting and Splitting
9. Cross-Namespace Routing
10. Backend Protocol Selection
11. HTTP Header Manipulation
12. Response and Request Timeouts
13. Authentication and Authorization
14. Cross-Origin Resource Sharing (CORS)
15. Rate Limiting and Traffic Policies
16. Custom Error Handling
17. WebSocket Support
18. TCP and UDP Routing
19. Multi-Cluster Gateway Patterns
20. Observability and Metrics
21. Security Considerations
22. Troubleshooting Gateway Deployments
23. Migration from Ingress to Gateway API
24. Reference Implementations and Providers




## 1. Gateway API Fundamentals

### What is the Gateway API?

The Gateway API is a collection of resources that model service networking in Kubernetes. Unlike the Ingress API, which offers a relatively simple model for HTTP routing, the Gateway API provides richer semantics through a set of interconnected custom resources that enable complex routing scenarios while maintaining clear separation of concerns.

Gateway API was designed to address limitations in the original Ingress resource:

- **Extensibility**: The Ingress API relied heavily on annotations, which varied across implementations
- **Role Separation**: Ingress mixed infrastructure and application concerns in a single resource
- **Expressiveness**: Many advanced routing features couldn't be expressed natively in Ingress
- **Multi-service Support**: Ingress wasn't designed for complex multi-service deployments

The Gateway API solves these problems through a carefully designed resource hierarchy, clear role boundaries, and first-class support for traffic management patterns that were previously challenging or impossible to implement.

### Maturity and Status

As of April 2025, the Gateway API has progressed beyond its initial experimental phase. Core components like GatewayClass, Gateway, and HTTPRoute have graduated to stable status (v1), while more specialized resources like TCPRoute, UDPRoute, and certain policy extensions remain in beta stages.

The Gateway API has been implemented by major service mesh and ingress controller providers, including Istio, Contour, Cilium, Kong, Traefik, NGINX, and AWS.

### Key Benefits

The Gateway API provides several advantages over traditional Ingress:

- **Role-oriented architecture**: Clear separation between infrastructure providers, cluster operators, and application developers
- **Portable configuration**: Standardized resources work across different implementations
- **Advanced traffic routing**: Native support for header-based matching, traffic splitting, and more
- **Cross-namespace capabilities**: Route traffic to services across namespace boundaries
- **Protocol support**: Beyond HTTP/HTTPS to TCP, UDP, TLS, and others
- **Policy attachment**: Modular approach to security, observability, and traffic management

## 2. Core Resources and Architecture

The Gateway API follows a role-oriented design with distinct resources owned by different personas:

### GatewayClass

GatewayClass is a cluster-scoped resource that defines a type of Gateway implementation offered by an infrastructure provider. It's similar to StorageClass in the way it represents an implementation that can be instantiated.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-gateway-implementation
spec:
  controllerName: example.com/gateway-controller
```

GatewayClass is typically created by cluster administrators or infrastructure providers.

### Gateway

Gateway represents the deployed instance of a GatewayClass, defining the actual listener configurations for accepting traffic. It specifies ports, protocols, and TLS settings.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: infrastructure
spec:
  gatewayClassName: example-gateway-implementation
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            environment: production
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: example-cert
        kind: Secret
    allowedRoutes:
      namespaces:
        from: All
```

Gateways are typically managed by cluster operators or network administrators.

### Route Resources

Route resources define the rules for matching and processing traffic. The primary route types include:

#### HTTPRoute

HTTPRoute configures HTTP routing including path-based routing, header matching, and traffic splitting.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: customer-route
  namespace: customer-portal
spec:
  parentRefs:
  - name: production-gateway
    namespace: infrastructure
  hostnames:
  - "customer.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: customer-api
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: customer-frontend
      port: 80
```

#### Other Route Types

- **TCPRoute**: For TCP traffic
- **UDPRoute**: For UDP traffic
- **TLSRoute**: For TLS traffic (SNI-based routing)
- **GRPCRoute**: For gRPC routing (beta)

These route resources are typically created and managed by application developers or service owners.

### Architecture Relationship

The relationship between these resources creates a clear separation of concerns:

1. **Infrastructure providers** implement and manage GatewayClasses
2. **Cluster operators** create and configure Gateways
3. **Application developers** define Routes to direct traffic to their services

This hierarchical model allows different teams to manage their own concerns without stepping on each other's toes, while enabling complex routing scenarios that were difficult with the Ingress API.

## 3. Basic Setup and Configuration

To get started with the Gateway API, you need to:

1. Install a Gateway API implementation
2. Create a GatewayClass (or use a pre-configured one)
3. Deploy a Gateway instance
4. Create Route resources to direct traffic

### Installing the Gateway API CRDs

First, ensure the Gateway API Custom Resource Definitions are installed in your cluster:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

### Selecting and Deploying an Implementation

Multiple providers implement the Gateway API. Here's an example using Contour:

```bash
# Install Contour with Gateway API support
kubectl apply -f https://projectcontour.io/quickstart/contour-gateway-provisioner.yaml

# Verify the GatewayClass is available
kubectl get gatewayclasses
```

### Creating a Gateway

Once the implementation is installed, create a Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
  namespace: gateway-system
spec:
  gatewayClassName: contour
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
```

Apply this configuration:

```bash
kubectl apply -f gateway.yaml
```

Wait for the Gateway to be provisioned:

```bash
kubectl -n gateway-system get gateway example-gateway
```

The `PROGRAMMED` condition should be `True` when the Gateway is ready.

### Creating a Simple HTTPRoute

Now deploy a sample application and route traffic to it:

```yaml
# Sample deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: nginx:latest
        ports:
        - containerPort: 80
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  namespace: default
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 80
---
# HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: sample-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "sample.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: sample-app
      port: 80
```

This simple setup creates a Gateway and an HTTPRoute that directs traffic for "sample.example.com" to a sample application.

### Verifying Routing

To test the routing:

```bash
# Get the Gateway's IP or hostname
kubectl -n gateway-system get gateway example-gateway -o jsonpath='{.status.addresses[0].value}'

# Test with curl (replace GATEWAY_IP with the actual IP)
curl -H "Host: sample.example.com" http://GATEWAY_IP/
```

## 4. TLS Termination and Certificate Management

The Gateway API provides flexible options for TLS termination, supporting both termination at the Gateway and passthrough modes.

### TLS Termination with Static Certificates

For basic TLS termination using a static certificate stored in a Kubernetes Secret:

```yaml
# Create a TLS secret
apiVersion: v1
kind: Secret
metadata:
  name: example-tls
  namespace: gateway-system
type: kubernetes.io/tls
data:
  tls.crt: base64-encoded-certificate
  tls.key: base64-encoded-private-key
---
# Configure Gateway with TLS
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: secure-gateway
  namespace: gateway-system
spec:
  gatewayClassName: contour
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: example-tls
        kind: Secret
    allowedRoutes:
      namespaces:
        from: All
```

### TLS Passthrough

For applications that need to handle TLS themselves:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: passthrough-gateway
  namespace: gateway-system
spec:
  gatewayClassName: contour
  listeners:
  - name: tls-passthrough
    port: 443
    protocol: TLS
    tls:
      mode: Passthrough
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: secure-backend
  namespace: default
spec:
  parentRefs:
  - name: passthrough-gateway
    namespace: gateway-system
  hostnames:
  - "secure.example.com"
  rules:
  - backendRefs:
    - name: tls-backend-service
      port: 8443
```

### Integration with Cert-Manager

For automated certificate management, you can integrate with cert-manager:

```yaml
# Create an Issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - name: example-gateway
            namespace: gateway-system

---
# Request a certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-cert
  namespace: gateway-system
spec:
  secretName: example-tls
  dnsNames:
  - example.com
  - www.example.com
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
```

When the certificate is issued, update the Gateway to use it:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: secure-gateway
  namespace: gateway-system
spec:
  gatewayClassName: contour
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: example-tls
        kind: Secret
    allowedRoutes:
      namespaces:
        from: All
```

## 5. HTTP Routing Essentials

The HTTPRoute resource is the primary method for configuring HTTP routing in the Gateway API. It provides powerful matching capabilities and flexible backend configuration.

### Basic Path-Based Routing

Route traffic based on URL paths:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: path-based-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "store.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /products
    backendRefs:
    - name: product-service
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /cart
    backendRefs:
    - name: cart-service
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /checkout
    backendRefs:
    - name: checkout-service
      port: 8080
```

### Path Types

The Gateway API supports different path matching types:

- **PathPrefix**: Matches URL paths that start with the specified prefix
- **Exact**: Matches the exact path
- **RegularExpression**: Matches paths based on a regular expression (implementation-specific)

Example with exact path matching:

```yaml
matches:
- path:
    type: Exact
    value: /api/v1/users
```

### Query Parameter Matching

Some implementations support matching on query parameters:

```yaml
matches:
- path:
    type: PathPrefix
    value: /search
  queryParams:
  - name: region
    value: europe
```

### Method-Based Routing

Route based on HTTP methods:

```yaml
matches:
- path:
    type: PathPrefix
    value: /api
  method: POST
```

### Hostname Matching

The `hostnames` field in HTTPRoute allows for routing based on the requested host:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: multi-hostname-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  - "api.example.org"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: api-service
      port: 8080
```

## 6. Header-Based Routing

Header-based routing allows for sophisticated traffic management based on HTTP request headers.

### Basic Header Matching

Route based on specific headers:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - headers:
      - name: "x-api-version"
        value: "v2"
    backendRefs:
    - name: api-v2-service
      port: 8080
  - matches:
    - headers:
      - name: "x-api-version"
        value: "v1"
    backendRefs:
    - name: api-v1-service
      port: 8080
  - backendRefs:
    - name: api-default-service
      port: 8080
```

### Header Matching Types

The Gateway API supports different header matching types:

- **Exact**: Default match type for exact header value
- **RegularExpression**: Match header values using regular expressions (implementation dependent)

Example with regular expression matching:

```yaml
matches:
- headers:
  - name: "user-agent"
    value: "Mozilla.*"
    type: RegularExpression
```

### Presence and Absence Matching

You can also match on the presence or absence of headers:

```yaml
# Match when a header exists
matches:
- headers:
  - name: "x-special-header"
    type: Exists

# Match when a header doesn't exist
matches:
- headers:
  - name: "x-special-header"
    type: NotExists
```

### Combining Multiple Headers

For more complex scenarios, combine multiple header conditions:

```yaml
matches:
- headers:
  - name: "x-api-version"
    value: "v2"
  - name: "x-region"
    value: "us-west"
```

This matches requests that have both headers with the specified values.

## 7. Traffic Weighting and Splitting

The Gateway API allows for sophisticated traffic splitting for use cases like canary releases, A/B testing, and blue-green deployments.

### Basic Traffic Splitting

Split traffic between two service versions based on weight:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app-v1
      port: 80
      weight: 90
    - name: app-v2
      port: 80
      weight: 10
```

This configuration sends 90% of traffic to app-v1 and 10% to app-v2.

### Gradual Canary Deployment

For a controlled rollout of a new version:

```yaml
# Initial state: 100% to stable
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gradual-rollout
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "service.example.com"
  rules:
  - backendRefs:
    - name: stable-service
      port: 80
      weight: 100

# Phase 1: 90/10 split
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gradual-rollout
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "service.example.com"
  rules:
  - backendRefs:
    - name: stable-service
      port: 80
      weight: 90
    - name: canary-service
      port: 80
      weight: 10

# Final state: 100% to new version
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gradual-rollout
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "service.example.com"
  rules:
  - backendRefs:
    - name: canary-service
      port: 80
      weight: 100
```

### Combining with Header-Based Routing

For cookie or header-based testing:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: targeted-canary
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "app.example.com"
  rules:
  # Test users always get the new version
  - matches:
    - headers:
      - name: "x-test-user"
        value: "true"
    backendRefs:
    - name: app-v2
      port: 80
  # Percentage-based split for everyone else
  - backendRefs:
    - name: app-v1
      port: 80
      weight: 95
    - name: app-v2
      port: 80
      weight: 5
```

## 8. Cross-Namespace Routing

The Gateway API allows routing traffic to services across different namespaces, with appropriate access controls.

### Enabling Cross-Namespace References

First, configure the Gateway to allow routes from specific namespaces:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: gateway-system
spec:
  gatewayClassName: example-gc
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            type: backend-services
```

### Routing to Services in Other Namespaces

With appropriate permissions, HTTPRoutes can reference services in other namespaces:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cross-namespace-route
  namespace: frontend
spec:
  parentRefs:
  - name: shared-gateway
    namespace: gateway-system
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      namespace: backend 
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80
```

### ReferenceGrant for Security

For security, explicit permission must be granted using a ReferenceGrant:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: frontend
  to:
  - group: ""
    kind: Service
```

This ReferenceGrant allows HTTPRoutes in the frontend namespace to reference Services in the backend namespace.

## 9. Header Manipulation

The Gateway API allows manipulation of request and response headers for various use cases like authentication, CORS, and custom headers.

### Adding Request Headers

Add headers to the request before forwarding to backends:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-modification
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        set:
        - name: "X-Source"
          value: "gateway-api"
        - name: "X-Environment"
          value: "production"
    backendRefs:
    - name: api-service
      port: 8080
```

### Response Header Manipulation

Similarly, modify response headers:

```yaml
filters:
- type: ResponseHeaderModifier
  responseHeaderModifier:
    set:
    - name: "Strict-Transport-Security"
      value: "max-age=31536000; includeSubDomains"
    add:
    - name: "X-Frame-Options"
      value: "DENY"
    remove:
    - "X-Powered-By"
```

### Header Manipulation for CORS

Configure Cross-Origin Resource Sharing with header modification:

```yaml
filters:
- type: ResponseHeaderModifier
  responseHeaderModifier:
    set:
    - name: "Access-Control-Allow-Origin"
      value: "https://example.com"
    - name: "Access-Control-Allow-Methods"
      value: "GET, POST, OPTIONS"
    - name: "Access-Control-Allow-Headers"
      value: "Content-Type, Authorization"
```

## 10. Request Redirection and URL Rewriting

The Gateway API provides built-in capabilities for URL redirection and path rewriting.

### URL Redirection

Implement redirects for URLs:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: redirect-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /old-path
    filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        hostname: "new.example.com"
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /new-path
        statusCode: 301
```

This configuration redirects requests from `http://example.com/old-path` to `https://new.example.com/new-path` with a 301 status code.

### Path Rewriting

Rewrite request paths before forwarding to backends:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/v1
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: api-service
      port: 8080
```

This rewrites `/api/v1/users` to `/users` before sending to the backend service.

### Hostname Rewriting

Change the host header for backends:

```yaml
filters:
- type: URLRewrite
  urlRewrite:
    hostname: "internal-service.svc.cluster.local"
```

## 11. Timeout and Retry Policies

Configure timeouts and retries for improved reliability:

### Setting Timeouts

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: timeout-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: RequestTimeout
      requestTimeout:
        requestTimeout: "10s"
    backendRefs:
    - name: api-service
      port: 8080
```

### Configuring Retries

```yaml
filters:
- type: RequestMirror
  requestMirror:
    backendRef:
      name: monitoring-service
      port: 8080
- type: ExtensionRef
  extensionRef:
    group: policy.gateway.networking.k8s.io
    kind: HTTPRetryPolicy
    name: api-retry-policy
```

With a corresponding retry policy:

```yaml
apiVersion: policy.gateway.networking.k8s.io/v1alpha1
kind: HTTPRetryPolicy
metadata:
  name: api-retry-policy
  namespace: default
spec:
  retryOn:
  - "5xx"
  numRetries: 3
  backoff:
    baseInterval: "100ms"
    maxInterval: "1s"
```

## 12. Traffic Mirroring

Mirror traffic to secondary backends for testing or monitoring:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mirror-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: RequestMirror
      requestMirror:
        backendRef:
          name: shadow-service
          port: 8080
    backendRefs:
    - name: api-service
      port: 8080
```

## 13. TCP and UDP Routing

Beyond HTTP, the Gateway API supports TCP and UDP protocols:

### TCP Routing

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tcp-gateway
  namespace: gateway-system
spec:
  gatewayClassName: example-gc
  listeners:
  - name: tcp
    port: 5432
    protocol: TCP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: database-route
  namespace: default
spec:
  parentRefs:
  - name: tcp-gateway
    namespace: gateway-system
  rules:
  - backendRefs:
    - name: postgresql
      port: 5432
```

### UDP Routing

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: udp-gateway
  namespace: gateway-system
spec:
  gatewayClassName: example-gc
  listeners:
  - name: udp
    port: 53
    protocol: UDP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: UDPRoute
metadata:
  name: dns-route
  namespace: default
spec:
  parentRefs:
  - name: udp-gateway
    namespace: gateway-system
  rules:
  - backendRefs:
    - name: dns-service
      port: 53
```

## 14. Security Considerations

### Network Policies

When implementing Gateway API resources, it's crucial to apply proper network policies to control traffic flow to and from your Gateway controllers. This layered security approach helps restrict unnecessary communication paths:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gateway-policy
  namespace: gateway-system
spec:
  podSelector:
    matchLabels:
      app: gateway
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
```

This policy allows external traffic to reach the Gateway controller on ports 80 and 443 while restricting the controller's outbound connections to only what's necessary.

### Authentication and Authorization

The Gateway API provides extension points for implementing authentication and authorization. Most implementations support integration with external identity providers:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: secure-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "secure.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: ExtensionRef
      extensionRef:
        group: gateway.networking.k8s.io
        kind: AuthFilter
        name: jwt-auth
    backendRefs:
    - name: api-service
      port: 8080
```

With a corresponding auth policy definition:

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha1
kind: AuthFilter
metadata:
  name: jwt-auth
  namespace: default
spec:
  type: JWT
  jwt:
    issuers:
    - uri: https://auth.example.com
    audiences:
    - api.example.com
```

### Role-Based Access Control (RBAC)

Define strict RBAC rules to control who can create and modify Gateway API resources:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: route-editor
  namespace: application
spec:
  rules:
  - apiGroups: ["gateway.networking.k8s.io"]
    resources: ["httproutes"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-route-editors
  namespace: application
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: route-editor
  apiGroup: rbac.authorization.k8s.io
```

### Secure Gateway Configuration

Follow these best practices for secure Gateway API deployments:

1. **Limit Route Attachment**: Use the `allowedRoutes` field to restrict which routes can attach to Gateways
2. **Implement TLS**: Always use TLS termination for production traffic
3. **Restrict Cross-Namespace References**: Use ReferenceGrants carefully to control service visibility
4. **Regular Updates**: Keep your Gateway controller implementation updated
5. **Resource Validation**: Use admission controllers to validate Gateway API resources

## 15. Rate Limiting and Traffic Policies

Rate limiting is essential for protecting services from overload, whether accidental or malicious. While the core Gateway API doesn't include rate limiting as a standard feature, most implementations offer this capability through extensions.

### Basic Rate Limiting

Using extension resources for rate limiting:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rate-limited-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: ExtensionRef
      extensionRef:
        group: policy.gateway.networking.k8s.io
        kind: RateLimitPolicy
        name: api-rate-limit
    backendRefs:
    - name: api-service
      port: 8080
```

With a corresponding rate limit policy:

```yaml
apiVersion: policy.gateway.networking.k8s.io/v1alpha1
kind: RateLimitPolicy
metadata:
  name: api-rate-limit
  namespace: default
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: rate-limited-route
  rates:
  - limit: 100
    duration: "1m"
    by:
    - source:
        type: RemoteIP
```

### Advanced Traffic Policies

Different implementations offer various traffic policy features. Here's an example using Istio's traffic policy features with Gateway API:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-auth-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: api-gateway
  rules:
  - from:
    - source:
        namespaces: ["frontend"]
    to:
    - operation:
        paths: ["/api/*"]
    when:
    - key: request.headers[x-api-key]
      values: ["valid-key"]
```

## 16. Custom Error Handling

Configure custom error responses for different HTTP status codes:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: custom-error-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "website.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    filters:
    - type: ExtensionRef
      extensionRef:
        group: policy.gateway.networking.k8s.io
        kind: ErrorPagePolicy
        name: custom-error-pages
    backendRefs:
    - name: website-service
      port: 80
```

With a corresponding error page policy:

```yaml
apiVersion: policy.gateway.networking.k8s.io/v1alpha1
kind: ErrorPagePolicy
metadata:
  name: custom-error-pages
  namespace: default
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: custom-error-route
  errorPages:
  - codes: [404]
    backendRef:
      name: error-service
      port: 8080
      path: /not-found
  - codes: [500, 502, 503, 504]
    backendRef:
      name: error-service
      port: 8080
      path: /server-error
```

## 17. WebSocket Support

The Gateway API supports WebSocket connections through HTTP upgrades:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: websocket-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "chat.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /ws
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        set:
        - name: "Connection"
          value: "Upgrade"
        - name: "Upgrade"
          value: "websocket"
    backendRefs:
    - name: websocket-service
      port: 8080
```

Most Gateway API implementations automatically detect and handle WebSocket upgrade requests when proper headers are present.

## 18. Multi-Cluster Gateway Patterns

As organizations adopt multi-cluster architectures, the Gateway API can facilitate traffic management across clusters.

### Shared Gateway Service

In this pattern, a dedicated Gateway service routes traffic to multiple clusters:

```yaml
# In the gateway cluster
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: multi-cluster-gateway
  namespace: gateway-system
spec:
  gatewayClassName: example-gc
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: multi-cluster-route
  namespace: gateway-system
spec:
  parentRefs:
  - name: multi-cluster-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /us-east
    backendRefs:
    - name: us-east-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /us-west
    backendRefs:
    - name: us-west-service
      port: 80
```

This approach requires service discovery mechanisms between clusters, often implemented with multi-cluster service mesh solutions like Istio or Cilium.

### Federated Gateways

Another pattern involves deploying Gateway instances in each cluster with consistent routing configurations:

```yaml
# Deployed in each cluster
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: regional-gateway
  namespace: gateway-system
spec:
  gatewayClassName: example-gc
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: regional-route
  namespace: default
spec:
  parentRefs:
  - name: regional-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: regional-service
      port: 80
```

This approach is often combined with global load balancing (like DNS-based routing) to direct users to the nearest regional gateway.

## 19. Observability and Metrics

Effective monitoring is crucial for Gateway API implementations. Most providers integrate with standard observability tools.

### Prometheus Integration

Configure Prometheus metrics collection:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gateway-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: gateway-controller
  endpoints:
  - port: metrics
    interval: 15s
```

### Common Metrics to Monitor

When operating Gateway API implementations, focus on these key metrics:

1. **Request rate**: Total requests per second
2. **Latency**: Response time distribution (p50, p90, p99)
3. **Error rate**: Percentage of 4xx and 5xx responses
4. **Connection metrics**: Active connections, connection rate
5. **TLS handshake metrics**: TLS negotiation times, failed handshakes
6. **Backend service health**: Success rate of backend services

### Distributed Tracing

Implement distributed tracing for comprehensive request flow visibility:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: traced-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        set:
        - name: "X-B3-Sampled"
          value: "1"
    backendRefs:
    - name: api-service
      port: 8080
```

Most Gateway implementations support OpenTelemetry or other tracing standards for end-to-end visibility of requests.

## 20. Troubleshooting Gateway Deployments

Effective troubleshooting of Gateway API implementations requires understanding the connection between resources and their operational status.

### Checking Gateway Status

Start by examining the Gateway resource status:

```bash
kubectl get gateway -n gateway-system example-gateway -o yaml
```

Look for conditions in the status field that indicate whether the Gateway has been successfully provisioned by the controller.

### Common Issues and Solutions

#### Routes Not Attaching to Gateway

**Symptoms**: Traffic not reaching services, HTTPRoute status shows no attached Gateways

**Troubleshooting**:

1. Check that the parentRef in the HTTPRoute matches the Gateway name and namespace
2. Verify that the Gateway's allowedRoutes configuration permits the Route's namespace
3. Inspect for any ReferenceGrant issues if using cross-namespace references

#### TLS Certificate Problems

**Symptoms**: HTTPS connections failing, TLS handshake errors

**Troubleshooting**:

1. Verify the certificate Secret exists in the correct namespace
2. Check the certificate's validity period and domain names
3. Confirm the Gateway's TLS configuration references the correct Secret

#### Service Connectivity Issues

**Symptoms**: 502 Bad Gateway or 503 Service Unavailable errors

**Troubleshooting**:

1. Verify the backend Service exists and has active endpoints
2. Check that the port specified in backendRefs matches the Service port
3. Use kubectl port-forward to test direct connectivity to the Service
4. Examine network policies that might be blocking traffic

### Debugging with Events and Logs

Gateway API controllers emit events for resources they manage:

```bash
# Check events for a Gateway
kubectl get events --field-selector involvedObject.name=example-gateway -n gateway-system

# Check events for an HTTPRoute
kubectl get events --field-selector involvedObject.name=example-route -n default
```

Controller logs often contain detailed information about configuration issues:

```bash
kubectl logs -n gateway-system deploy/gateway-controller -f
```

## 21. Migration from Ingress to Gateway API

Organizations moving from Ingress to Gateway API should follow a methodical approach.

### Mapping Concepts

|Ingress Concept|Gateway API Equivalent|
|---|---|
|Ingress|HTTPRoute|
|IngressClass|GatewayClass|
|Annotations|Native fields or policy extensions|
|Backend|backendRefs|
|Path rules|matches with path conditions|
|Host rules|hostnames|

### Example Migration

**Original Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**Equivalent Gateway API resources:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
  namespace: gateway-system
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: example.com
    tls:
      mode: Terminate
      certificateRefs:
      - name: example-tls
        kind: Secret
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-route
  namespace: default
spec:
  parentRefs:
  - name: example-gateway
    namespace: gateway-system
  hostnames:
  - "example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: api-service
      port: 80
```

### Migration Strategy

1. **Start with non-critical services**: Begin with low-risk services to gain experience
2. **Deploy both systems in parallel**: Run Gateway API alongside existing Ingress during transition
3. **Gradually shift traffic**: Use traffic splitting to slowly migrate users
4. **Monitor carefully**: Compare performance metrics between old and new routing

## 22. Reference Implementations and Providers

The Gateway API is implemented by numerous controllers and service meshes. Here's an overview of major implementations as of April 2025:

### Contour

Contour is a high-performance ingress controller that was an early adopter of the Gateway API:

```bash
# Install Contour with Gateway API support
kubectl apply -f https://projectcontour.io/quickstart/contour-gateway-provisioner.yaml
```

### Istio

Istio's service mesh capabilities extend Gateway API with advanced traffic management:

```bash
# Install Istio with Gateway API enabled
istioctl install --set profile=default --set values.pilot.env.PILOT_ENABLE_GATEWAY_API=true
```

### NGINX Gateway Fabric

Built on NGINX, this implementation brings enterprise-grade features to Gateway API:

```bash
# Install NGINX Gateway Fabric
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.0.0/nginx-gateway-fabric.yaml
```

### Cilium

Cilium offers eBPF-powered networking with Gateway API support:

```bash
# Install Cilium with Gateway API
helm install cilium cilium/cilium --namespace kube-system --set gatewayAPI.enabled=true
```

### Implementation Feature Comparison

|Feature|Contour|Istio|NGINX|Cilium|
|---|---|---|---|---|
|Gateway v1 support|✓|✓|✓|✓|
|HTTPRoute v1 support|✓|✓|✓|✓|
|TCPRoute support|✓|✓|✓|✓|
|UDPRoute support|✗|✓|✗|✓|
|GRPCRoute support|✓|✓|✗|✓|
|Policy attachments|Partial|Full|Partial|Full|
|Multi-cluster support|✗|✓|✗|✓|

## 23. Future Directions

The Gateway API continues to evolve with new features and capabilities planned:

### Service Mesh Integration

Enhanced integration between Gateway API and service mesh features:

```yaml
# Example of future mesh integration
apiVersion: gateway.networking.k8s.io/v1alpha1
kind: MeshGateway
metadata:
  name: mesh-gateway
  namespace: mesh-system
spec:
  gatewayClassName: service-mesh-gateway
  listeners:
  - protocol: MESH
    port: 15443
    tls:
      mode: Passthrough
```

### Expanded Policy Framework

A comprehensive policy framework for centralized management:

```yaml
# Example of future policy attachment
apiVersion: policy.gateway.networking.k8s.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: api-security
  namespace: default
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: api-route
  rules:
  - securitySchemes:
    - type: OAuth2
      oauth2:
        tokenEndpoint: https://auth.example.com/token
        scopes: ["api:read"]
```

### Extended Protocol Support

Expanded protocol support beyond current offerings:

```yaml
# Example of future QUIC/HTTP3 support
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: modern-gateway
  namespace: gateway-system
spec:
  gatewayClassName: example-gc
  listeners:
  - name: http3
    port: 443
    protocol: HTTP3
    tls:
      mode: Terminate
      certificateRefs:
      - name: example-tls
```

