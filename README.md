# Master Blueprint

This document covers the complete lifecycle of your multi-node cluster, including the "battle-tested" fixes we implemented for the IP changes and Traefik routing.
## Note: All these IP's are Dynamically assigned by the Router, suggest to make it static IP's.

## 🏗️ Phase 1: Cluster Installation

### 1.1 Dep System (The Master/Brain) [Deployment System]
**IP:** `10.84.106.126`

Run this to install the K3s server with a specific "Advertise IP" to prevent future mismatch:

```bash
curl -sfL https://get.k3s.io | sh -s - server --tls-san 10.84.106.126 --write-kubeconfig-mode 644
```

Get the Join Token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### 1.2 Dev System (The Worker/Muscle) [Development System]
**IP:** `10.84.106.68`

Run this to join the worker to the master:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.84.106.126:6443 \
K3S_TOKEN=<YOUR_TOKEN_FROM_STEP_1.1> sh -
```

## 🛠️ Phase 2: Configuration & YAMLs

Run these on the **Dep System**.

### 2.1 Node Labeling & Taints

```bash
# Taint the master so it only runs backend/heavy tasks
sudo kubectl taint nodes pawan-aspire-a311-45 hardware=high-performance:NoSchedule

# Label the worker for the frontend
sudo kubectl label node pawan-aspire-a715-42g tier=frontend
```

### 2.2 The Code (YAML Files)

The project includes the following configuration files:
- `backend-deploy.yaml`
- `frontend-deploy.yaml`
- `services.yaml`
- `middleware.yaml`
- `ingress.yaml`

(See separate files for content)

## 🧪 Phase 3: Testing

### Internal Pod Testing (from Dev System):

```bash
# Test Frontend Pod directly
curl 10.42.1.3:80

# Test API Pod across the network tunnel
curl 10.42.0.9:80
```

### External Ingress Testing (from Browser or CLI):

```bash
# Test Frontend via Ingress
curl -i http://10.84.106.126/

# Test API via Ingress (Verified with Middleware)
curl -i http://10.84.106.126/api
```

## 🔍 Phase 4: Issues Faced & Solved

| Issue | Cause | Fix |
| :--- | :--- | :--- |
| **Node NotReady** | Master IP changed from 10.90... to 10.84... | Updated `K3S_URL` on Agent and restarted `k3s-agent`. |
| **Connection Refused** | K3s API crashed after IP change | Ran `systemctl restart k3s` on Master and updated TLS certificates. |
| **API Call Hung (Timeout)** | Firewall blocking the internal tunnel | Ran `sudo ufw allow 8472/udp` on both systems. |
| **404 Not Found (/api)** | Traefik kept the `/api` prefix | Created a Middleware with `stripPrefix` to clean the URL. |
| **CRD Version Error** | Used `traefik.containo.us` (old) | Used `kubectl api-resources` to find the correct `traefik.io/v1alpha1`. |

# Quick Reference Checklist

## 1. Installation & Cluster Setup

**On Dep System (Master)**
```bash
# Install K3s Server with specific IP
curl -sfL https://get.k3s.io | sh -s - server --tls-san 10.84.106.126 --write-kubeconfig-mode 644

# Get the join token for the worker
sudo cat /var/lib/rancher/k3s/server/node-token
```

**On Dev System (Worker)**
```bash
# Join the cluster as an Agent (Replace <TOKEN> with the output from above)
curl -sfL https://get.k3s.io | K3S_URL=https://10.84.106.126:6443 K3S_TOKEN=<TOKEN> sh -
```

## 🛡️ 2. Firewall & Networking (Crucial)

Run these on **BOTH Systems** to ensure the nodes can talk over the physical cable.

```bash
# Allow the Kubernetes Control Plane
sudo ufw allow 6443/tcp

# Allow the Flannel VXLAN tunnel (Internal Pod Communication)
sudo ufw allow 8472/udp

# Allow the internal Pod/Service networks
sudo ufw allow from 10.42.0.0/16
sudo ufw allow from 10.43.0.0/16
```

## 🏗️ 3. Deployment & Logic (Dep System)

Use these to apply the YAML files we created.

```bash
# 1. Label the Worker Node
sudo kubectl label node pawan-aspire-a715-42g tier=frontend

# 2. Taint the Master Node
sudo kubectl taint nodes pawan-aspire-a311-45 hardware=high-performance:NoSchedule

# 3. Apply the YAMLs
sudo kubectl apply -f backend-deploy.yaml
sudo kubectl apply -f frontend-deploy.yaml
sudo kubectl apply -f services.yaml
sudo kubectl apply -f middleware.yaml
sudo kubectl apply -f ingress.yaml
```

## 🔍 4. Verification & Debugging (Dep System)

Use these to check if the "Engine" is running correctly.

```bash
# Check if Nodes are 'Ready'
sudo kubectl get nodes

# Check if Pods are 'Running' and see which node they are on
sudo kubectl get pods -o wide

# Check if Services have Endpoints (Internal IPs)
sudo kubectl get endpoints

# Check if Ingress has picked up the IP address
sudo kubectl get ingress

# Check if the Middleware exists
sudo kubectl get middleware
```

## 🧪 5. Testing the Endpoints (Dev System / CLI)

Use these to confirm the website is working before opening the browser.

```bash
# Test the Frontend (Should return <h1>Hello...</h1>)
curl -i http://10.84.106.126/

# Test the API (Should return "Hello from Node.js API")
curl -i http://10.84.106.126/api

# Debug: Test the Pod IP directly (from Dev System)
curl 10.42.0.9:80
```

## 🚨 6. Troubleshooting / Recovery

If the IP changes again or things break:

```bash
# Restart the Master (Dep)
sudo systemctl restart k3s

# Restart the Agent (Dev)
sudo systemctl restart k3s-agent

# See live logs if something fails
sudo journalctl -u k3s -f
sudo journalctl -u k3s-agent -f

# Check API resources if YAMLs give 'Mapping not found' errors
sudo kubectl api-resources | grep -i middleware
```

## 🌍 7. Browser Testing [These don't work, but I left them here for reference]

Frontend: http://10.84.106.126/

Backend: http://10.84.106.126/api
