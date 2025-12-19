# Bare-Metal Kubernetes — Setup Explanation

When running Kubernetes on **bare metal**, you don’t get built-in LoadBalancers like you would on cloud providers. To solve this, you install two key components:

Here’s a more polished and professional version of your K3S installation instructions:

## K3S Installation

To avoid conflicts with nginx-ingress, it is recommended to disable Traefik during K3S installation or remove it from an existing deployment.

### Installing K3S Without Traefik

You can install K3S without Traefik using the following command:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -s -
k3s server &
```

### Removing Traefik From an Existing K3S Installation

If K3S is already installed, Traefik can be removed as follows:

```bash
sudo rm -rf /var/lib/rancher/k3s/server/manifests/traefik.yaml
helm uninstall traefik traefik-crd -n kube-system
sudo systemctl restart k3s
```

This ensures that Traefik is completely removed and prevents potential conflicts with the nginx-ingress controller.

## 1. MetalLB — LoadBalancer for Bare Metal

Cloud platforms automatically assign external IPs to Kubernetes `LoadBalancer` services.
Bare metal clusters **don’t**, so **MetalLB** fills this gap by providing:

* A pool of external IPs managed inside your local network
* Automatic assignment of those IPs to `LoadBalancer` services
* ARP/NDP announcements so your network knows where traffic should go

### **Installed with Helm:**

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb
```

### **Configuration Explained:**

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: default
spec:
  addresses:
  - <IP>/32
  autoAssign: true
```

* **IPAddressPool**: Defines which IPs MetalLB is allowed to hand out
* `<IP>/32`: A *single* IP address for your LoadBalancer service
* `autoAssign: true`: MetalLB will automatically pick this IP when a service requests one

## 3. HAProxy TCP LoadBalancer for Bare-Metal

For bare-metal Kubernetes clusters, TCP services (like Minecraft or Velocity proxies) require an ingress controller that can handle raw TCP traffic. **HAProxy Ingress** is commonly used in combination with MetalLB to expose these services.

### TCP Service CRD

HAProxy supports defining TCP frontends via a Kubernetes CRD. Example for a Minecraft service:

```yaml
apiVersion: ingress.v1.haproxy.org/v1
kind: TCP
metadata:
  name: minecraft-tcp-service
  annotations:
    ingress.class: haproxy
spec:
  - name: minecraft-tcp
    frontend:
      name: minecraft-tcp-frontend
      tcplog: true
      binds:
        - name: proxy-bind
          port: 19132
    service:
      name: proxy-server
      port: 19132
```

* `frontend.name`: Logical name for the HAProxy frontend.
* `binds.port`: Port HAProxy listens on externally.
* `service.name` / `service.port`: The Kubernetes service that receives the forwarded TCP traffic.
* `tcplog: true`: Enables TCP logging for easier debugging.

### Helm Configuration for TCP Ports

Alternatively, you can define TCP ports in the HAProxy Helm chart values:

```yaml
controller:
  name: controller
  service:
    type: LoadBalancer
    tcpPorts:
      - name: velocity-proxy
        port: 19132      # External port on HAProxy
        targetPort: 19132 # Port on backend service
        nodePort: 30000   # Optional NodePort
        protocol: TCP
```

* `port`: Port exposed via HAProxy.
* `targetPort`: Port the backend service listens on.
* `nodePort`: Optional; allows access on all nodes.
* `protocol`: TCP (or UDP if required).

### Applying the Configuration

Update HAProxy with the Helm chart:

```bash
helm upgrade haproxy-kubernetes-ingress -f tcp-ports.yml haproxytech/kubernetes-ingress --namespace haproxy-controller
```

This will apply your TCP mappings and ensure HAProxy routes external traffic to your backend services!

### How It Works with MetalLB

1. **MetalLB** assigns a public IP to the HAProxy `LoadBalancer` service.
2. **HAProxy** listens on the specified TCP ports.
3. Incoming traffic is forwarded to the appropriate Kubernetes Service and Pod.

This setup enables bare-metal clusters to expose TCP services to the network using a combination of MetalLB and HAProxy.
