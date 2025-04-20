
# Setting Up DNS Resolution for Kubernetes Kind Cluster

This document explains how to set up local DNS resolution using dnsmasq for a Kubernetes Kind cluster, allowing services to be accessed via custom domain names without modifying `/etc/hosts`.

## Overview

When working with Kubernetes locally, accessing services via domain names can be challenging. This guide demonstrates how to configure dnsmasq to automatically resolve all domains ending with `.kind.cluster` to your Kind node's IP address, simplifying service access through Ingress resources.

## Prerequisites

- Linux system with systemd
- Docker and Kind installed
- Administrative (sudo) privileges
- Basic understanding of Kubernetes networking concepts

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
NODEIP=$(docker network inspect kind -f '{{(index .IPAM.Config 0).Gateway}}')
echo "kind node IP = $NODEIP"
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

## Using Your DNS Configuration with Kubernetes

Now that DNS resolution is set up, you can create Ingress resources that use hostnames with the `.kind.cluster` suffix:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fitness-hero-ingress
  namespace: frontend
spec:
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

With this configuration, accessing `http://fitness-hero.kind.cluster` in your browser will connect you to the `fitness-hero-svc` service in your Kind cluster.

## Troubleshooting

If you encounter issues with DNS resolution:

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
    
4. Ensure your browser isn't using its own DNS resolver (like DoH - DNS over HTTPS)
    

## Conclusion

By using dnsmasq with a wildcard domain configuration, you've simplified access to services running in your Kind cluster. This approach eliminates the need to constantly update `/etc/hosts` entries when working with different services.