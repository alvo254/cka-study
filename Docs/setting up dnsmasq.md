# Setting Up DNS Resolution for Kubernetes Kind Cluster

This document explains how to set up local DNS resolution using dnsmasq for a Kubernetes Kind cluster, allowing services to be accessed via custom domain names without modifying `/etc/hosts`. It also covers how to configure TLS termination for secure HTTPS access.

## Overview

When working with Kubernetes locally, accessing services via domain names can be challenging. This guide demonstrates how to configure dnsmasq to automatically resolve all domains ending with `.kind.cluster` to your Kind node's IP address, simplifying service access through Ingress resources. Additionally, it covers setting up TLS termination to enable HTTPS for your local services.

## Prerequisites

- Linux system with systemd
- Docker and Kind installed
- Administrative (sudo) privileges
- Basic understanding of Kubernetes networking concepts
- Ingress controller installed in your cluster (nginx-ingress is used in examples)

## Step-by-Step Configuration

### 1. Prepare systemd-resolved to Work with dnsmasq

First, configure systemd-resolved to use dnsmasq as the local DNS resolver:

```bash
# Create configuration directory if it doesn't exist
sudo mkdir -p /etc/systemd/resolved.conf.d

# Create configuration file to direct DNS queries to dnsmasq
cat <<'EOF' | sudo tee /etc/systemd/resolved.conf.d/dnsmasq.conf
[Resolve]
DNS=127.0.0.1
DNSStubListener=no
EOF

# Restart systemd-resolved to apply changes
sudo systemctl restart systemd-resolved
```

This configuration tells systemd-resolved to:

- Use 127.0.0.1 (localhost) as the DNS server, where dnsmasq will be listening
- Disable the stub listener to avoid port conflicts with dnsmasq

### 2. Install dnsmasq

If not already installed:

```bash
sudo apt update
sudo apt install dnsmasq
```

### 3. Find Your Kind Cluster's IP Address

The Kind cluster runs in Docker containers. To find the IP address of the cluster's network gateway:

```bash
# Find the node's bridge IP (kind default network)
NODEIP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kind-control-plane)
echo "kind node IP = $NODEIP"
```

```fish
set NODEIP (docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kind-control-plane)
```

This command:

1. Uses `docker network inspect` to examine the "kind" network
2. Extracts the gateway IP address using Go template formatting
3. Stores it in the NODEIP variable (typically something like 172.18.0.1)

### 4. Configure dnsmasq with Modular Configuration Files

dnsmasq supports a modular configuration approach through the `/etc/dnsmasq.d/` directory. Any files placed in this directory with a `.conf` extension are loaded as part of the configuration.

```bash
# Add a wildcard *.kind.cluster to dnsmasq
echo "address=/.kind.cluster/${NODEIP}" | sudo tee -a /etc/dnsmasq.d/kind-local.conf
```

```fish
echo "address=/.kind.cluster/$NODEIP"| sudo tee /etc/dnsmasq.d/kind-local.conf >/dev/null
```

This command:

1. Creates a file called `kind-local.conf` in the dnsmasq.d directory
2. Adds a configuration entry that resolves all domains ending with `.kind.cluster` to your Kind node's IP

### 5. Restart dnsmasq to Apply Changes

```bash
sudo systemctl restart dnsmasq
```

### 6. Verify Configuration

Test that DNS resolution is working by pinging any domain with the `.kind.cluster` suffix:

```bash
ping test.kind.cluster
```

If configured correctly, this will resolve to your Kind node's IP address.

## Understanding dnsmasq's Modular Configuration

dnsmasq supports a modular configuration approach through the `/etc/dnsmasq.d/` directory. This design offers several benefits:

### How Modular Configuration Works

1. The main configuration file (`/etc/dnsmasq.conf`) typically includes this line:
    
    ```
    conf-dir=/etc/dnsmasq.d/,*.conf
    ```
    
2. This directive tells dnsmasq to:
    
    - Look in the `/etc/dnsmasq.d/` directory
    - Load any file ending with `.conf` as a configuration fragment
    - Process these files in alphabetical order

### Benefits of This Approach

- **Separation of concerns**: Different aspects of configuration can be placed in different files
- **Easier management**: Configuration for different purposes can be added or removed without editing the main file
- **Safer updates**: Package updates won't overwrite your custom configurations
- **Better organization**: You can name files to reflect their purpose (e.g., `kind-local.conf` for Kind cluster settings)

### Examples of Modular Configuration

You can create multiple configuration files for different purposes:

- `local-domains.conf` - For personal development domains
- `kind-local.conf` - For Kind cluster domains
- `special-options.conf` - For specific dnsmasq options

## Setting Up TLS Termination

To enable HTTPS access to your services, you need to configure TLS termination at the Ingress level. This involves creating a self-signed certificate (for development) and configuring your Ingress resource to use it.

### 1. Create a Self-Signed Certificate

For development purposes, you can create a self-signed certificate:

```bash
# Create directory for certificates
mkdir -p ~/certs

# Generate a private key
openssl genrsa -out ~/certs/kind-cluster.key 2048

# Create a self-signed certificate
openssl req -x509 -new -nodes -key ~/certs/kind-cluster.key -sha256 -days 365 \
  -out ~/certs/kind-cluster.crt -subj "/CN=*.kind.cluster" \
  -addext "subjectAltName = DNS:*.kind.cluster,DNS:kind.cluster"
```

### 2. Create a TLS Secret in Kubernetes

Upload your certificate to Kubernetes as a Secret:

```bash
# Create the namespace if it doesn't exist
kubectl create namespace frontend

# Create a TLS secret from your certificate files
kubectl create secret tls kind-cluster-tls -n frontend \
  --key ~/certs/kind-cluster.key \
  --cert ~/certs/kind-cluster.crt
```

### 3. Configure Ingress with TLS

Update your Ingress resource to use the TLS certificate:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fitness-hero-ingress
  namespace: frontend
  annotations:
    # Optional: Force HTTPS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - fitness-hero.kind.cluster
    secretName: kind-cluster-tls
  rules:
  - host: fitness-hero.kind.cluster
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fitness-hero-svc
            port:
              number: 3000
```

### 4. Apply the Ingress Configuration

```bash
# Save the above YAML to fitness-hero-ingress.yaml
kubectl apply -f fitness-hero-ingress.yaml
```

### 5. Trust the Self-Signed Certificate (Optional)

For local development, you may want to add your certificate to your system's trust store:

#### On Ubuntu/Debian:

```bash
sudo cp ~/certs/kind-cluster.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

#### On macOS:

```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/certs/kind-cluster.crt
```

#### For browsers only:

Most browsers allow you to manually add exceptions for self-signed certificates when you visit the site.

## Using Your DNS and TLS Configuration with Kubernetes

Now that DNS resolution and TLS are set up, you can access your service securely:

```
https://fitness-hero.kind.cluster
```

Your browser will establish a secure connection to your service running in the Kind cluster.

## Troubleshooting

If you encounter issues with DNS resolution or TLS:

### DNS Issues:

1. Verify that dnsmasq is running:
    
    ```bash
    sudo systemctl status dnsmasq
    ```
    
2. Check dnsmasq logs for errors:
    
    ```bash
    sudo journalctl -u dnsmasq
    ```
    
3. Test DNS resolution using dig:
    
    ```bash
    dig test.kind.cluster @127.0.0.1
    ```
    

### TLS Issues:

1. Check that the secret was created correctly:
    
    ```bash
    kubectl get secret kind-cluster-tls -n frontend -o yaml
    ```
    
2. Examine ingress controller logs:
    
    ```bash
    kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
    ```
    
3. Verify the ingress resource:
    
    ```bash
    kubectl get ingress -n frontend
    kubectl describe ingress fitness-hero-ingress -n frontend
    ```
    
4. Test TLS connection:
    
    ```bash
    curl -vk https://fitness-hero.kind.cluster
    ```
    

## Conclusion

By using dnsmasq with a wildcard domain configuration and setting up TLS termination, you've created a secure and convenient local development environment for your Kubernetes services. This approach eliminates the need to constantly update `/etc/hosts` entries and provides a more production-like environment with HTTPS support.