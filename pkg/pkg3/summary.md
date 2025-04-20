1. Verify your KIND cluster

  
```
kubectl cluster-info

kubectl get nodes -o wide
```


  

Make sure all nodes are Ready and you know the control-plane node’s Docker container name (kind-control-plane) and IP.

  

2. Add the ingress-nginx Helm repository

  
```
helm repo add ingress-nginx https://ingress-nginx.github.io/ingress-nginx

helm repo update
```


  

3. Install NGINX Ingress Controller with hostPort enabled

  

This command installs the controller into its own namespace and binds ports 80 and 443 on each node’s network interface.

  
```
helm install ingress-nginx ingress-nginx/ingress-nginx \
			--namespace ingress-nginx --create-namespace \
			--set controller.hostPort.enabled=true \
			--set controller.hostPort.http=80 \
			--set controller.hostPort.https=443
```

# Check that the namespace exists

```
kubectl get ns ingress-nginx
```
  
# Ensure the controller pod(s) are Running

```
kubectl -n ingress-nginx get pods -o wide
```

  
# Confirm hostPort binding on the node

```

ss -tlnp | grep ':80 ' # should show nginx-ingress controller

ss -tlnp | grep ':443 '

```

  

4. Configure dnsmasq for local wildcard DNS

  

Determine your control-plane node IP:


```
set NODEIP (docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kind-control-plane)
```

  
Create (or overwrite) a dnsmasq rule to map *.kind.cluster to the node IP:


```
echo "address=/.kind.cluster/$NODEIP" | sudo tee /etc/dnsmasq.d/kind-local.conf
```

  

Restart dnsmasq and flush DNS caches:


```
sudo systemctl restart dnsmasq

sudo resolvectl flush-caches
```

  
Verify DNS resolution  

```
dig +short example.kind.cluster @127.0.0.1 # should return the control-plane IP
```

  
5. Test your Ingress

  

Apply an Ingress resource (e.g., fitness-hero-ingress).

  

Ensure DNS resolves to the node and port 80 is reachable:

  
```

curl http://fitness-hero.kind.cluster/

```
  

If you receive a valid HTTP response from your backend service, the setup is successful!