# Comprehensive Guide to Kubernetes Ingress Controllers

## Introduction

Ingress controllers in Kubernetes provide sophisticated HTTP/HTTPS routing capabilities that go far beyond simple service exposure. They serve as intelligent gateways, managing external access to services in a cluster, and offer a wide range of traffic management features.

This document provides a detailed exploration of Ingress controller capabilities, from basic setup to advanced configurations, with practical examples using the popular NGINX Ingress Controller.

## Table of Contents

- [Comprehensive Guide to Kubernetes Ingress Controllers](#comprehensive-guide-to-kubernetes-ingress-controllers)
  - [Introduction](#introduction)
  - [Table of Contents](#table-of-contents)
  - [Core Concepts](#core-concepts)
  - [Basic Setup](#basic-setup)
    - [Installing NGINX Ingress Controller](#installing-nginx-ingress-controller)
    - [Basic Ingress Resource](#basic-ingress-resource)
  - [TLS Termination](#tls-termination)
    - [Creating a TLS Certificate Secret](#creating-a-tls-certificate-secret)
    - [Configuring TLS in Ingress](#configuring-tls-in-ingress)
    - [Wildcard Certificates](#wildcard-certificates)
    - [Automatic Certificate Management with cert-manager](#automatic-certificate-management-with-cert-manager)
  - [Path-Based Routing](#path-based-routing)
    - [Path Types](#path-types)
  - [Host-Based Routing](#host-based-routing)
  - [URL Rewriting and Redirects](#url-rewriting-and-redirects)
    - [URL Path Rewriting](#url-path-rewriting)
    - [External URL Redirects](#external-url-redirects)
    - [Internal Path Redirects](#internal-path-redirects)
  - [Header Manipulation](#header-manipulation)
    - [Adding Custom Headers](#adding-custom-headers)
    - [Setting Security Headers](#setting-security-headers)
    - [Modifying or Removing Headers](#modifying-or-removing-headers)
  - [Rate Limiting](#rate-limiting)
    - [Rate Limiting by IP](#rate-limiting-by-ip)
  - [Authentication](#authentication)
    - [Basic Authentication](#basic-authentication)
    - [External OAuth2 Authentication](#external-oauth2-authentication)
  - [Session Affinity and Sticky Sessions](#session-affinity-and-sticky-sessions)
  - [Traffic Splitting and Canary Deployments](#traffic-splitting-and-canary-deployments)
    - [Header-Based Canary](#header-based-canary)
  - [Backend Protocol Conversion](#backend-protocol-conversion)
  - [WebSockets Support](#websockets-support)
  - [Customizing Timeouts and Buffer Sizes](#customizing-timeouts-and-buffer-sizes)
  - [CORS Configuration](#cors-configuration)
  - [Custom Error Pages](#custom-error-pages)
  - [Monitoring and Metrics](#monitoring-and-metrics)
  - [Troubleshooting](#troubleshooting)
    - [Checking Ingress Controller Logs](#checking-ingress-controller-logs)
    - [Verify Ingress Configuration](#verify-ingress-configuration)
    - [Testing connectivity](#testing-connectivity)
    - [Common Issues](#common-issues)

## Core Concepts

An Ingress controller in Kubernetes consists of three main components:

1. **Ingress Resources**: Kubernetes API objects that define rules for routing external HTTP(S) traffic to internal services
2. **Ingress Controller**: The implementation that fulfills the rules defined in Ingress resources
3. **Load Balancer**: External component that directs traffic to the Ingress controller

Common implementations include:
- NGINX Ingress Controller (most popular)
- Traefik
- HAProxy
- Kong
- Istio Gateway

This guide focuses on the NGINX Ingress Controller but many concepts apply across implementations.

## Basic Setup

### Installing NGINX Ingress Controller

```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx

# Or using kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml
```

### Basic Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  namespace: default
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

This configuration routes all traffic for `myapp.example.com` to the `myapp-service` on port 80.

## TLS Termination

TLS termination allows your Ingress controller to handle HTTPS traffic and encryption/decryption.

### Creating a TLS Certificate Secret

```bash
# Generate a self-signed certificate (for development)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=myapp.example.com"

# Create a Kubernetes Secret from the certificate files
kubectl create secret tls myapp-tls \
  --key tls.key \
  --cert tls.crt
```

### Configuring TLS in Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"  # Force HTTP to HTTPS redirect
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### Wildcard Certificates

You can use wildcard certificates to secure multiple subdomains:

```yaml
spec:
  tls:
  - hosts:
    - "*.example.com"
    secretName: wildcard-example-tls
```

### Automatic Certificate Management with cert-manager

For production environments, you can use cert-manager to automatically obtain and manage certificates from Let's Encrypt:

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml

# Create a ClusterIssuer for Let's Encrypt
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

Then configure your Ingress to use cert-manager:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secured-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-cert
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

## Path-Based Routing

Path-based routing directs traffic to different services based on the URL path.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8081
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Path Types

Kubernetes supports three `pathType` values:

- `Prefix`: Matches based on a URL path prefix split by `/`. Matching is case-sensitive.
- `Exact`: Matches the URL path exactly and is case-sensitive.
- `ImplementationSpecific`: Matching is based on the ingress controller implementation.

```yaml
paths:
- path: /exact-path
  pathType: Exact
  backend:
    service:
      name: exact-service
      port:
        number: 80
```

## Host-Based Routing

Host-based routing directs traffic based on the hostname in the request.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8081
```

## URL Rewriting and Redirects

### URL Path Rewriting

URL rewriting modifies the path before forwarding the request to the backend service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

In this example, a request to `/api/users` will be rewritten to `/users` before being sent to the backend service.

### External URL Redirects

You can configure redirects to external URLs:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: redirect-ingress
  annotations:
    nginx.ingress.kubernetes.io/permanent-redirect: "https://new.example.com"
spec:
  rules:
  - host: old.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dummy-service
            port:
              number: 80
```

### Internal Path Redirects

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-redirect-ingress
  annotations:
    nginx.ingress.kubernetes.io/permanent-redirect-code: "301"
    nginx.ingress.kubernetes.io/permanent-redirect: "/new-path"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /old-path
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

## Header Manipulation

### Adding Custom Headers

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: header-ingress
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Custom-Header: custom-value";
      more_set_headers "X-Another-Header: another-value";
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### Setting Security Headers

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: security-headers-ingress
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Content-Security-Policy: default-src 'self'";
spec:
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-service
            port:
              number: 80
```

### Modifying or Removing Headers

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: modify-headers-ingress
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      # Remove Server header
      more_clear_headers "Server";
      
      # Modify existing headers
      proxy_hide_header X-Powered-By;
      more_set_headers "X-Powered-By: Kubernetes";
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

## Rate Limiting

Rate limiting helps protect your services from abuse or overload.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
    nginx.ingress.kubernetes.io/enable-global-auth: "false"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

This limits clients to 10 requests per second and 5 simultaneous connections.

### Rate Limiting by IP

```yaml
annotations:
  nginx.ingress.kubernetes.io/limit-rps: "10"
  nginx.ingress.kubernetes.io/limit-whitelist: "192.168.0.1/24,172.17.0.0/16"
```

## Authentication

### Basic Authentication

1. Create an auth file:
```bash
htpasswd -c auth admin
kubectl create secret generic basic-auth --from-file=auth
```

2. Configure the Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-service
            port:
              number: 80
```

### External OAuth2 Authentication

You can integrate with OAuth providers like Auth0, Okta, or Keycloak:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: oauth-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "https://oauth2-proxy.example.com/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://oauth2-proxy.example.com/oauth2/start?rd=$escaped_request_uri"
    nginx.ingress.kubernetes.io/auth-response-headers: "X-Auth-Request-User, X-Auth-Request-Email"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

This requires setting up an OAuth2 Proxy service separately.

## Session Affinity and Sticky Sessions

Sticky sessions ensure that a client always connects to the same backend pod.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sticky-ingress
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "STICKYSESSION"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

## Traffic Splitting and Canary Deployments

Canary deployments allow you to route a percentage of traffic to a new version.

1. Create a canary deployment and service:
```bash
kubectl create deployment myapp-canary --image=myapp:v2
kubectl expose deployment myapp-canary --port=80
```

2. Configure canary routing:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-canary
            port:
              number: 80
```

This routes 20% of traffic to the canary service. The remaining 80% goes to the service defined in the main ingress.

### Header-Based Canary

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
  nginx.ingress.kubernetes.io/canary-by-header-value: "true"
```

With this configuration, requests with the header `X-Canary: true` will be routed to the canary service.

## Backend Protocol Conversion

You can use HTTPS to communicate with backend services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ssl-backend-ingress
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-backend
            port:
              number: 443
```

## WebSockets Support

Enable WebSockets for specific paths:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  rules:
  - host: socket.example.com
    http:
      paths:
      - path: /ws
        pathType: Prefix
        backend:
          service:
            name: websocket-service
            port:
              number: 8080
```

## Customizing Timeouts and Buffer Sizes

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: timeout-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"  # in seconds
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"     # in seconds
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"     # in seconds
    nginx.ingress.kubernetes.io/proxy-body-size: "8m"        # max request body size
    nginx.ingress.kubernetes.io/proxy-buffer-size: "128k"    # buffer size for reading client request header
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

## CORS Configuration

Enable Cross-Origin Resource Sharing:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cors-ingress
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, PUT, POST, DELETE, PATCH, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://trusted-origin.com"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization"
    nginx.ingress.kubernetes.io/cors-max-age: "86400"  # in seconds (24 hours)
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

## Custom Error Pages

Configure custom error pages:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: error-pages-ingress
  annotations:
    nginx.ingress.kubernetes.io/default-backend: error-pages-service
    nginx.ingress.kubernetes.io/custom-http-errors: "404,500,502,503,504"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

You'll need to create an error-pages service that serves custom error pages.

## Monitoring and Metrics

The NGINX Ingress Controller provides metrics in Prometheus format:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: metrics-ingress
  annotations:
    nginx.ingress.kubernetes.io/enable-metrics: "true"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

Access metrics at:
```
http://<ingress-controller-service>:10254/metrics
```

Common metrics include:
- Request counters by status code
- Latency measurements
- Connection statistics
- Resource usage (CPU, memory)

## Troubleshooting

### Checking Ingress Controller Logs

```bash
# Get pod name
INGRESS_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

# View logs
kubectl logs -n ingress-nginx $INGRESS_POD

# Enable debug logging
kubectl edit configmap -n ingress-nginx ingress-nginx-controller
# Set "error-log-level: debug" in the data section
```

### Verify Ingress Configuration

```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
```

### Testing connectivity

```bash
# Test from inside the cluster
kubectl run curl --image=curlimages/curl -i --tty -- sh
curl -v http://myapp-service.default.svc.cluster.local

# Test ingress connectivity
curl -v -H "Host: myapp.example.com" http://<ingress-controller-ip>
```

### Common Issues

1. **Certificate issues**: Check if secrets exist and are correctly referenced
2. **Service not found**: Verify service name and port configuration
3. **Path routing issues**: Check path patterns and path types
4. **Authentication failures**: Verify auth secrets and configurations
5. **Timeout errors**: Check timeouts in annotations and backend service response times

---

This guide covers the most important features and capabilities of Kubernetes Ingress controllers, with a focus on the NGINX implementation. By leveraging these features, you can create sophisticated traffic routing rules, implement secure access patterns, and optimize the performance of your web applications running in Kubernetes.