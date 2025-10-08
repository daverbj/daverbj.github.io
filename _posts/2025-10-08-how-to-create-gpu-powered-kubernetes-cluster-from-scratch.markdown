---
layout: post
title: "How to Create a GPU-Powered Kubernetes Cluster from Scratch for Your AI Workload"
date: 2025-10-08 10:00:00 +0300
categories: kubernetes ai infrastructure
author: Debashis Banerjee
tags: [kubernetes, gpu, ai, mlops, infrastructure, nvidia, cuda]
---

# Kubernetes Cluster Setup Guide

## Cluster Configuration
- **Control Plane:** 10.20.21.57
- **Node1 (GPU):** 10.31.20.35 (NVIDIA L40)
- **Node2 (GPU):** 10.31.20.36 (NVIDIA L40)

---

## Prerequisites

Before starting, ensure all nodes have:
- Ubuntu 20.04/22.04 or similar Linux distribution
- At least 2 CPU cores and 2GB RAM
- Network connectivity between all nodes
- Root or sudo access

---

## Part 1: Configure All Nodes

Run these steps on **all three machines** (control plane + both worker nodes).

### 1.1 Set Hostnames and Hosts File

**On Control Plane (10.20.21.57):**
```bash
sudo hostnamectl set-hostname k8s-control
```

**On Node1 (10.31.20.35):**
```bash
sudo hostnamectl set-hostname k8s-node1
```

**On Node2 (10.31.20.36):**
```bash
sudo hostnamectl set-hostname k8s-node2
```

**On All Nodes:**
```bash
sudo tee -a /etc/hosts <<EOF
10.20.21.57 k8s-control
10.31.20.35 k8s-node1
10.31.20.36 k8s-node2
EOF
```

### 1.2 Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 1.3 Load Required Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 1.4 Configure Sysctl Parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 1.5 Configure Firewall Ports

**On Control Plane:**
```bash
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10259/tcp
sudo ufw allow 10257/tcp
sudo ufw allow 8472/udp
sudo ufw allow 8285/udp
sudo ufw reload
```

**On Worker Nodes:**
```bash
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp
sudo ufw allow 8472/udp
sudo ufw allow 8285/udp
sudo ufw reload
```

### 1.6 Install Containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 1.7 Install Kubernetes Components

```bash
# Add Kubernetes repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install packages
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Part 2: Initialize Control Plane

Run these steps **only on the control plane (10.20.21.57)**.

### 2.1 Initialize Kubernetes

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.20.21.57
```

**Important:** After initialization completes, you'll see a `kubeadm join` command. **Copy and save this command** - you'll need it to join the worker nodes!

### 2.2 Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 2.3 Install Pod Network (Flannel)

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### 2.4 Verify Control Plane

```bash
kubectl get nodes
kubectl get pods -A
```

Wait until the control plane node shows as "Ready" and all system pods are running.

---

## Part 3: Configure GPU Support on Worker Nodes

Run these steps **only on Node1 and Node2 (10.31.20.35 and 10.31.20.36)**.

### 3.1 Verify NVIDIA Drivers

```bash
nvidia-smi
```

If this command fails, install the drivers:

```bash
sudo apt-get update
sudo apt-get install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot
```

### 3.2 Install NVIDIA Container Toolkit

```bash
# Add NVIDIA repository
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Install toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# Configure containerd for NVIDIA
sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart containerd
```

---

## Part 4: Join Worker Nodes to Cluster

Run these steps **on both worker nodes (Node1 and Node2)**.

### 4.1 Join the Cluster

Use the `kubeadm join` command you saved from Step 2.1. It looks like:

```bash
sudo kubeadm join 10.20.21.57:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### 4.2 If You Lost the Join Command

Run this on the control plane to generate a new one:

```bash
kubeadm token create --print-join-command
```

---

## Part 5: Install NVIDIA Device Plugin

Run these steps **on the control plane**.

### 5.1 Deploy NVIDIA Device Plugin

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.5/nvidia-device-plugin.yml
```

### 5.2 Verify GPU Nodes

```bash
kubectl get nodes
kubectl describe nodes k8s-node1 | grep nvidia.com/gpu
kubectl describe nodes k8s-node2 | grep nvidia.com/gpu
```

You should see `nvidia.com/gpu: 1` (or more) in the capacity section.

---

## Part 6: Verify Your Cluster

Run these commands **on the control plane**.

### 6.1 Check Cluster Status

```bash
# Check all nodes
kubectl get nodes

# Check all pods
kubectl get pods -A

# Check GPU availability
kubectl get nodes -o json | jq '.items[].status.capacity'
```

### 6.2 Test GPU Workload

Create a test pod:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  containers:
  - name: cuda-container
    image: nvcr.io/nvidia/cuda:12.0.0-base-ubuntu22.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
  nodeSelector:
    kubernetes.io/hostname: k8s-node1
EOF
```

Wait for the pod to be running:

```bash
kubectl get pods -w
```

Test GPU access:

```bash
kubectl exec -it gpu-test -- nvidia-smi
```

If you see GPU information, your cluster is working correctly!

### 6.3 Clean Up Test Pod

```bash
kubectl delete pod gpu-test
```

---

## Troubleshooting

### Nodes Not Joining
- Check firewall rules are applied correctly
- Verify network connectivity: `ping k8s-control` from worker nodes
- Check token validity: `kubeadm token list` on control plane

### Pods Not Starting
- Check containerd status: `sudo systemctl status containerd`
- View pod logs: `kubectl logs <pod-name> -n <namespace>`
- Describe pod: `kubectl describe pod <pod-name>`

### GPU Not Detected
- Verify NVIDIA drivers: `nvidia-smi` on worker nodes
- Check device plugin pods: `kubectl get pods -n kube-system | grep nvidia`
- Check containerd config: `sudo cat /etc/containerd/config.toml | grep nvidia`

---

## Part 7: Install Helm Package Manager

Helm is the package manager for Kubernetes, making it easy to install and manage applications.

### 7.1 Install Helm (Run on Control Plane)

**Method 1: Using the official script (Recommended)**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Method 2: Manual installation**

```bash
# Download Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

**Method 3: Using apt (Ubuntu/Debian)**

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### 7.2 Verify Helm Installation

```bash
helm version
```

You should see output showing the Helm version (e.g., v3.13.x or later).

### 7.3 Add Common Helm Repositories

```bash
# Add Bitnami repository (popular charts)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Add stable repository
helm repo add stable https://charts.helm.sh/stable

# Add NVIDIA GPU Operator repository (useful for GPU management)
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia

# Add Prometheus community (monitoring)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Add Grafana (visualization)
helm repo add grafana https://grafana.github.io/helm-charts

# Update repositories
helm repo update
```

### 7.4 Verify Repositories

```bash
helm repo list
```

### 7.5 Search for Charts

```bash
# Search all repositories
helm search repo nginx

# Search a specific repository
helm search repo bitnami/mysql
```

### 7.6 Install Your First Application (Example: Nginx)

```bash
# Create a namespace
kubectl create namespace demo

# Install nginx
helm install my-nginx bitnami/nginx --namespace demo

# Check the installation
helm list -n demo
kubectl get pods -n demo
```

### 7.7 Useful Helm Commands

```bash
# List all installed releases
helm list -A

# Get release status
helm status my-nginx -n demo

# Get release values
helm get values my-nginx -n demo

# Upgrade a release
helm upgrade my-nginx bitnami/nginx --namespace demo

# Rollback a release
helm rollback my-nginx 1 -n demo

# Uninstall a release
helm uninstall my-nginx -n demo
```

### 7.8 Install Helm on Your Local Machine (Optional)

If you want to manage your cluster from your local machine:

**Linux/macOS:**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Windows (using Chocolatey):**
```powershell
choco install kubernetes-helm
```

**Windows (using Scoop):**
```powershell
scoop install helm
```

Then copy your kubeconfig from the control plane:

```bash
# On control plane
cat ~/.kube/config

# On your local machine
mkdir -p ~/.kube
# Paste the content into ~/.kube/config
```

### 7.9 Example: Deploy a GPU Application with Helm

Since you have GPU nodes, here's an example deploying a CUDA application:

```bash
# Create values file for GPU deployment
cat <<EOF > gpu-app-values.yaml
resources:
  limits:
    nvidia.com/gpu: 1
nodeSelector:
  kubernetes.io/hostname: k8s-node1
EOF

# Install with custom values
helm install my-gpu-app bitnami/tensorflow --values gpu-app-values.yaml -n demo
```

### 7.10 Clean Up Demo Resources

```bash
helm uninstall my-nginx -n demo
kubectl delete namespace demo
```

---

## Part 8: Install Prometheus Monitoring Stack

Deploy Prometheus and Grafana for comprehensive cluster monitoring, including GPU metrics.

### 8.1 Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

### 8.2 Install kube-prometheus-stack

This installs Prometheus, Grafana, Alertmanager, and various exporters:

```bash
# Add prometheus-community repo (if not already added)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install the complete monitoring stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123
```

**Note:** Change `admin123` to a secure password!

### 8.3 Verify Installation

```bash
# Check all pods are running
kubectl get pods -n monitoring

# Wait for all pods to be ready (this may take 2-3 minutes)
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=grafana -n monitoring --timeout=300s
```

### 8.4 Access Grafana Dashboard

**Method 1: Port Forward (Quick Access)**

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

Then open your browser to: `http://localhost:3000`
- **Username:** admin
- **Password:** admin123 (or whatever you set)

**Method 2: NodePort (Persistent Access)**

```bash
# Set NodePort with a specific port (must be in range 30000-32767)
kubectl patch svc prometheus-grafana -n monitoring -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "targetPort": 3000, "nodePort": 30100, "protocol": "TCP", "name": "http-web"}]}}'

# Verify the NodePort
kubectl get svc prometheus-grafana -n monitoring
```

Access via: `http://10.20.21.57:30100` (or any node IP)

**Note:** You can change `30100` to any port between 30000-32767 that's not already in use. To check available ports:

```bash
# See all NodePorts in use
kubectl get svc -A | grep NodePort
```

**Method 3: LoadBalancer (if you have a load balancer)**

```bash
kubectl patch svc prometheus-grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
```

### 8.5 Access Prometheus UI

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

Open: `http://localhost:9090`

### 8.6 Install NVIDIA GPU Exporter for GPU Monitoring

To monitor your NVIDIA L40 GPUs:

```bash
# Create GPU exporter DaemonSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-dcgm-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: nvidia-dcgm-exporter
  template:
    metadata:
      labels:
        app: nvidia-dcgm-exporter
    spec:
      nodeSelector:
        nvidia.com/gpu: "true"
      containers:
      - name: nvidia-dcgm-exporter
        image: nvcr.io/nvidia/k8s/dcgm-exporter:3.1.8-3.1.5-ubuntu20.04
        securityContext:
          runAsNonRoot: false
          runAsUser: 0
        volumeMounts:
        - name: pod-gpu-resources
          readOnly: true
          mountPath: /var/lib/kubelet/pod-resources
      volumes:
      - name: pod-gpu-resources
        hostPath:
          path: /var/lib/kubelet/pod-resources
---
apiVersion: v1
kind: Service
metadata:
  name: nvidia-dcgm-exporter
  namespace: monitoring
  labels:
    app: nvidia-dcgm-exporter
spec:
  type: ClusterIP
  ports:
  - name: metrics
    port: 9400
    targetPort: 9400
    protocol: TCP
  selector:
    app: nvidia-dcgm-exporter
EOF
```

### 8.7 Create ServiceMonitor for GPU Metrics

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nvidia-dcgm-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: nvidia-dcgm-exporter
  endpoints:
  - port: metrics
    interval: 30s
EOF
```

### 8.8 Verify GPU Metrics in Prometheus

1. Port-forward Prometheus: `kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090`
2. Open `http://localhost:9090`
3. Go to "Graph" and search for: `DCGM_FI_DEV_GPU_UTIL`
4. You should see GPU utilization metrics

### 8.9 Import GPU Dashboard in Grafana

1. Access Grafana (see step 8.4)
2. Login with admin credentials
3. Go to **Dashboards** → **Import**
4. Enter dashboard ID: **12239** (NVIDIA DCGM Exporter Dashboard)
5. Click **Load**
6. Select **Prometheus** as the data source
7. Click **Import**

You now have a beautiful GPU monitoring dashboard!

### 8.10 Useful Pre-built Dashboards

Import these dashboard IDs in Grafana:

- **12239** - NVIDIA DCGM Exporter Dashboard
- **315** - Kubernetes Cluster Monitoring
- **1860** - Node Exporter Full
- **7249** - Kubernetes Cluster
- **6417** - Kubernetes Cluster Monitoring (via Prometheus)

### 8.11 Create a Custom Alert for GPU

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-alerts
  namespace: monitoring
spec:
  groups:
  - name: gpu
    interval: 30s
    rules:
    - alert: HighGPUTemperature
      expr: DCGM_FI_DEV_GPU_TEMP > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "GPU temperature is high"
        description: "GPU {{ \$labels.gpu }} on {{ \$labels.instance }} has temperature {{ \$value }}°C"
    - alert: HighGPUUtilization
      expr: DCGM_FI_DEV_GPU_UTIL > 95
      for: 10m
      labels:
        severity: info
      annotations:
        summary: "GPU utilization is very high"
        description: "GPU {{ \$labels.gpu }} on {{ \$labels.instance }} has utilization {{ \$value }}%"
EOF
```

### 8.12 View Alerts

Access Alertmanager:

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
```

Open: `http://localhost:9093`

### 8.13 Common Monitoring Queries

Use these in Prometheus or Grafana:

```promql
# GPU Utilization
DCGM_FI_DEV_GPU_UTIL

# GPU Memory Usage
DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_FREE * 100

# GPU Temperature
DCGM_FI_DEV_GPU_TEMP

# GPU Power Usage
DCGM_FI_DEV_POWER_USAGE

# Node CPU Usage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Node Memory Usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Pod CPU Usage
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod, namespace)

# Pod Memory Usage
sum(container_memory_working_set_bytes) by (pod, namespace)
```

### 8.14 Upgrade Prometheus Stack

To upgrade the monitoring stack:

```bash
helm repo update
helm upgrade prometheus prometheus-community/kube-prometheus-stack -n monitoring
```

### 8.15 Uninstall Prometheus (if needed)

```bash
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```

---

## Part 9: Install NGINX Ingress Controller

NGINX Ingress allows you to access your services (like Prometheus and Grafana) via domain names instead of port-forwarding.

### 9.1 Install NGINX Ingress Controller

```bash
# Add nginx ingress repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install nginx ingress controller
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

**Note:** We're using NodePort with ports 30080 (HTTP) and 30443 (HTTPS) for easy access.

### 9.2 Verify Installation

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Wait for the ingress controller pod to be ready:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### 9.3 Configure /etc/hosts (On Your Local Machine)

Add these entries to access services via domain names:

**Linux/Mac:**
```bash
sudo tee -a /etc/hosts <<EOF
10.20.21.57 grafana.local
10.20.21.57 prometheus.local
10.20.21.57 alertmanager.local
EOF
```

**Windows:**
Edit `C:\Windows\System32\drivers\etc\hosts` and add:
```
10.20.21.57 grafana.local
10.20.21.57 prometheus.local
10.20.21.57 alertmanager.local
```

### 9.4 Create Ingress for Grafana

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: grafana.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
EOF
```

### 9.5 Create Ingress for Prometheus

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: prometheus.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-kube-prometheus-prometheus
            port:
              number: 9090
EOF
```

### 9.6 Create Ingress for Alertmanager

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alertmanager-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: alertmanager.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-kube-prometheus-alertmanager
            port:
              number: 9093
EOF
```

### 9.7 Verify Ingress Resources

```bash
kubectl get ingress -n monitoring
```

You should see three ingress resources with addresses assigned.

### 9.8 Access Your Services

Now you can access your monitoring services via browser:

- **Grafana:** http://grafana.local:30080
- **Prometheus:** http://prometheus.local:30080
- **Alertmanager:** http://alertmanager.local:30080

**Login credentials for Grafana:**
- Username: `admin`
- Password: `admin123` (or whatever you set in Part 8)

### 9.9 Optional: Enable HTTPS with Self-Signed Certificate

Generate a self-signed certificate:

```bash
# Create certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=*.local/O=monitoring"

# Create Kubernetes secret
kubectl create secret tls monitoring-tls \
  --cert=tls.crt --key=tls.key \
  -n monitoring

# Clean up local files
rm tls.key tls.crt
```

Update Grafana ingress to use TLS:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - grafana.local
    secretName: monitoring-tls
  rules:
  - host: grafana.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
EOF
```

Access via HTTPS: https://grafana.local:30443

(You'll need to accept the self-signed certificate warning in your browser)

### 9.10 Optional: Use LoadBalancer Instead of NodePort

If you have a LoadBalancer available (like MetalLB):

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer
```

Then access services on port 80/443 directly without the NodePort numbers.

### 9.11 Add Basic Authentication (Optional)

Secure your services with basic auth:

```bash
# Install htpasswd utility
sudo apt-get install apache2-utils

# Create auth file (username: admin, you'll be prompted for password)
htpasswd -c auth admin

# Create secret
kubectl create secret generic basic-auth --from-file=auth -n monitoring

# Clean up
rm auth
```

Update Prometheus ingress with authentication:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
spec:
  ingressClassName: nginx
  rules:
  - host: prometheus.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-kube-prometheus-prometheus
            port:
              number: 9090
EOF
```

### 9.12 View Ingress Logs

```bash
# View nginx ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=100 -f
```

### 9.13 Common Ingress Troubleshooting

```bash
# Check ingress controller status
kubectl get pods -n ingress-nginx

# Describe ingress resource
kubectl describe ingress grafana-ingress -n monitoring

# Check ingress controller service
kubectl get svc -n ingress-nginx

# Test from within the cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -H "Host: grafana.local" http://nginx-ingress-ingress-nginx-controller.ingress-nginx.svc.cluster.local
```

### 9.14 Uninstall NGINX Ingress (if needed)

```bash
helm uninstall nginx-ingress -n ingress-nginx
kubectl delete namespace ingress-nginx
```

---

## Part 10: Backup and Disaster Recovery Strategy

Protecting your cluster data is critical. This section covers backup strategies and disaster recovery procedures.

### 10.1 Understanding What to Backup

**Critical Components:**
1. **etcd** - Cluster state and configuration (most critical)
2. **Persistent Volumes** - Application data
3. **Kubernetes Resources** - Deployments, services, configmaps, secrets
4. **Helm Releases** - Application configurations

### 10.2 Install Velero for Cluster Backup

Velero is the industry-standard backup solution for Kubernetes.

**Prerequisites: Install MinIO for backup storage**

```bash
# Create minio namespace
kubectl create namespace minio

# Install MinIO
helm repo add minio https://charts.min.io/
helm install minio minio/minio \
  --namespace minio \
  --set rootUser=minioadmin \
  --set rootPassword=minioadmin123 \
  --set mode=standalone \
  --set persistence.size=50Gi \
  --set service.type=NodePort \
  --set service.nodePort=30900

# Wait for MinIO to be ready
kubectl wait --for=condition=ready pod -l app=minio -n minio --timeout=300s
```

**Create MinIO bucket for backups:**

```bash
# Port forward to access MinIO
kubectl port-forward -n minio svc/minio 9001:9001 &

# Access MinIO console at http://localhost:9001
# Login: minioadmin / minioadmin123
# Create a bucket named "velero"
# Or use mc client:

# Install mc client
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configure mc
mc alias set myminio http://localhost:9001 minioadmin minioadmin123
mc mb myminio/velero
```

**Install Velero CLI:**

```bash
# Download Velero
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/
rm -rf velero-v1.12.0-linux-amd64*

# Verify installation
velero version --client-only
```

**Create credentials file:**

```bash
cat > credentials-velero <<EOF
[default]
aws_access_key_id = minioadmin
aws_secret_access_key = minioadmin123
EOF
```

**Install Velero in cluster:**

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.minio.svc:9000

# Clean up credentials file
rm credentials-velero
```

**Verify Velero installation:**

```bash
kubectl get pods -n velero
velero backup-location get
```

### 10.3 Backup etcd Directly (Control Plane)

etcd contains all cluster state. This is the most critical backup.

**Create etcd backup script:**

```bash
cat > /usr/local/bin/backup-etcd.sh <<'EOF'
#!/bin/bash
BACKUP_DIR="/var/backups/etcd"
DATE=$(date +%Y%m%d-%H%M%S)
ETCD_ENDPOINTS="https://127.0.0.1:2379"
ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"
ETCD_CA="/etc/kubernetes/pki/etcd/ca.crt"

mkdir -p $BACKUP_DIR

ETCDCTL_API=3 etcdctl \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CA \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  snapshot save $BACKUP_DIR/etcd-snapshot-$DATE.db

# Keep only last 7 days of backups
find $BACKUP_DIR -name "etcd-snapshot-*.db" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/etcd-snapshot-$DATE.db"
EOF

sudo chmod +x /usr/local/bin/backup-etcd.sh
```

**Run manual backup:**

```bash
sudo /usr/local/bin/backup-etcd.sh
```

**Setup automatic daily backups (cron):**

```bash
# Run backup daily at 2 AM
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/backup-etcd.sh >> /var/log/etcd-backup.log 2>&1") | crontab -
```

### 10.4 Create Full Cluster Backup with Velero

**Backup entire cluster:**

```bash
velero backup create full-cluster-backup --wait
```

**Backup specific namespace:**

```bash
velero backup create monitoring-backup --include-namespaces monitoring --wait
```

**Backup with specific resources:**

```bash
velero backup create app-backup \
  --include-namespaces default \
  --include-resources deployments,services,configmaps,secrets \
  --wait
```

**Schedule automatic backups:**

```bash
# Daily backup at 1 AM
velero schedule create daily-backup --schedule="0 1 * * *"

# Weekly full backup on Sunday at 3 AM
velero schedule create weekly-backup --schedule="0 3 * * 0"

# Backup monitoring namespace daily
velero schedule create monitoring-daily --schedule="0 2 * * *" --include-namespaces monitoring
```

**List backups:**

```bash
velero backup get
velero schedule get
```

### 10.5 Backup Kubernetes Resources (Manual Method)

**Backup all resources to YAML:**

```bash
# Create backup directory
mkdir -p ~/k8s-backups/$(date +%Y%m%d)
cd ~/k8s-backups/$(date +%Y%m%d)

# Backup all namespaces
kubectl get namespaces -o yaml > namespaces.yaml

# Backup all resources in each namespace
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  echo "Backing up namespace: $ns"
  mkdir -p $ns
  
  # Backup deployments
  kubectl get deployments -n $ns -o yaml > $ns/deployments.yaml
  
  # Backup services
  kubectl get services -n $ns -o yaml > $ns/services.yaml
  
  # Backup configmaps
  kubectl get configmaps -n $ns -o yaml > $ns/configmaps.yaml
  
  # Backup secrets
  kubectl get secrets -n $ns -o yaml > $ns/secrets.yaml
  
  # Backup persistent volume claims
  kubectl get pvc -n $ns -o yaml > $ns/pvc.yaml
  
  # Backup ingresses
  kubectl get ingress -n $ns -o yaml > $ns/ingress.yaml
done

# Backup persistent volumes (cluster-wide)
kubectl get pv -o yaml > persistent-volumes.yaml

# Backup storage classes
kubectl get sc -o yaml > storage-classes.yaml

# Compress backup
cd ..
tar -czf k8s-backup-$(date +%Y%m%d).tar.gz $(date +%Y%m%d)
```

**Automate with cron:**

```bash
# Save the script
cat > /usr/local/bin/backup-k8s-resources.sh <<'EOF'
#!/bin/bash
BACKUP_ROOT=~/k8s-backups
DATE=$(date +%Y%m%d)
BACKUP_DIR=$BACKUP_ROOT/$DATE

mkdir -p $BACKUP_DIR
cd $BACKUP_DIR

kubectl get namespaces -o yaml > namespaces.yaml

for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  mkdir -p $ns
  kubectl get all,cm,secret,pvc,ingress -n $ns -o yaml > $ns/all-resources.yaml
done

kubectl get pv,sc -o yaml > cluster-resources.yaml

cd $BACKUP_ROOT
tar -czf k8s-backup-$DATE.tar.gz $DATE
rm -rf $DATE

# Keep only last 30 days
find $BACKUP_ROOT -name "k8s-backup-*.tar.gz" -mtime +30 -delete
EOF

chmod +x /usr/local/bin/backup-k8s-resources.sh

# Schedule daily at 3 AM
(crontab -l 2>/dev/null; echo "0 3 * * * /usr/local/bin/backup-k8s-resources.sh >> /var/log/k8s-backup.log 2>&1") | crontab -
```

### 10.6 Backup Helm Releases

```bash
# List all helm releases
helm list -A

# Backup helm release values
mkdir -p ~/helm-backups
for release in $(helm list -A -q); do
  namespace=$(helm list -A | grep $release | awk '{print $2}')
  helm get values $release -n $namespace > ~/helm-backups/$release-values.yaml
  helm get manifest $release -n $namespace > ~/helm-backups/$release-manifest.yaml
done
```

### 10.7 Disaster Recovery Procedures

**Scenario 1: Restore from Velero Backup**

```bash
# List available backups
velero backup get

# Restore entire cluster
velero restore create --from-backup full-cluster-backup --wait

# Restore specific namespace
velero restore create --from-backup monitoring-backup --include-namespaces monitoring --wait

# Check restore status
velero restore get
velero restore describe <restore-name>
```

**Scenario 2: Restore etcd from Snapshot**

```bash
# Stop kube-apiserver
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore etcd snapshot
sudo ETCDCTL_API=3 etcdctl snapshot restore /var/backups/etcd/etcd-snapshot-YYYYMMDD-HHMMSS.db \
  --data-dir=/var/lib/etcd-restore \
  --name=k8s-control \
  --initial-cluster=k8s-control=https://10.20.21.57:2380 \
  --initial-advertise-peer-urls=https://10.20.21.57:2380

# Backup current etcd data
sudo mv /var/lib/etcd /var/lib/etcd-backup

# Move restored data
sudo mv /var/lib/etcd-restore /var/lib/etcd

# Start kube-apiserver
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Wait for cluster to be ready
kubectl get nodes
```

**Scenario 3: Rebuild Cluster from Scratch**

If you need to completely rebuild:

1. **Restore etcd on new control plane:**
   - Follow etcd restore procedure above
   
2. **Rejoin worker nodes:**
   - Reset nodes: `sudo kubeadm reset`
   - Join with new token: `kubeadm token create --print-join-command`
   
3. **Reinstall CNI and add-ons:**
   - Apply Flannel
   - Install NVIDIA device plugin
   - Install monitoring stack
   - Install ingress controller

4. **Restore applications with Velero:**
   ```bash
   velero restore create --from-backup full-cluster-backup
   ```

### 10.8 Offsite Backup Strategy

**Copy backups to remote location:**

```bash
# Create script for remote sync
cat > /usr/local/bin/sync-backups.sh <<'EOF'
#!/bin/bash
REMOTE_HOST="backup-server.example.com"
REMOTE_USER="backup"
REMOTE_PATH="/backups/k8s-cluster"

# Sync etcd backups
rsync -avz --delete /var/backups/etcd/ $REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH/etcd/

# Sync resource backups
rsync -avz --delete ~/k8s-backups/ $REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH/resources/

# Sync helm backups
rsync -avz --delete ~/helm-backups/ $REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH/helm/
EOF

chmod +x /usr/local/bin/sync-backups.sh

# Run after backups complete (4 AM)
(crontab -l 2>/dev/null; echo "0 4 * * * /usr/local/bin/sync-backups.sh >> /var/log/backup-sync.log 2>&1") | crontab -
```

### 10.9 Backup Verification

**Test restores regularly:**

```bash
# Create a test namespace
kubectl create namespace backup-test

# Deploy a test application
kubectl create deployment nginx --image=nginx -n backup-test

# Create backup
velero backup create test-backup --include-namespaces backup-test --wait

# Delete namespace
kubectl delete namespace backup-test

# Restore
velero restore create --from-backup test-backup --wait

# Verify
kubectl get all -n backup-test

# Clean up
kubectl delete namespace backup-test
velero backup delete test-backup
```

### 10.10 Monitoring Backup Health

**Create alerts for backup failures:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: velero-backup-alerts
  namespace: velero
spec:
  groups:
  - name: velero
    interval: 30s
    rules:
    - alert: VeleroBackupFailed
      expr: velero_backup_failure_total > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Velero backup has failed"
        description: "Backup {{ \$labels.schedule }} has failed"
    - alert: VeleroBackupTooOld
      expr: time() - velero_backup_last_successful_timestamp{schedule!=""} > 86400
      for: 30m
      labels:
        severity: warning
      annotations:
        summary: "Velero backup is too old"
        description: "Last successful backup for {{ \$labels.schedule }} was more than 24 hours ago"
EOF
```

### 10.11 Backup Checklist

**Daily:**
- ✓ etcd snapshot (automated via cron)
- ✓ Velero scheduled backups
- ✓ Verify backup completion

**Weekly:**
- ✓ Test restore procedure
- ✓ Verify offsite backup sync
- ✓ Review backup storage usage

**Monthly:**
- ✓ Full disaster recovery drill
- ✓ Update documentation
- ✓ Review and update backup retention policies

### 10.12 Backup Best Practices

1. **3-2-1 Rule:** 3 copies, 2 different media types, 1 offsite
2. **Test restores regularly** - Untested backups are not backups
3. **Monitor backup jobs** - Set up alerts for failures
4. **Document procedures** - Keep recovery documentation up to date
5. **Encrypt sensitive data** - Especially for offsite backups
6. **Automate everything** - Reduce human error

### 10.13 Quick Recovery Commands Reference

```bash
# List all backups
velero backup get

# Restore entire cluster
velero restore create --from-backup <backup-name>

# Restore specific namespace
velero restore create --from-backup <backup-name> --include-namespaces <namespace>

# Check restore status
velero restore describe <restore-name>

# Restore etcd
sudo ETCDCTL_API=3 etcdctl snapshot restore <snapshot-file>

# Generate new join token
kubeadm token create --print-join-command
```

---

## Part 11: Detaching Nodes from the Cluster

Use these steps when you need to remove a worker node from the cluster (for maintenance, decommissioning, or reconfiguration).

### 7.1 Drain the Node (Run on Control Plane)

First, safely evict all pods from the node:

```bash
# For node1
kubectl drain k8s-node1 --ignore-daemonsets --delete-emptydir-data

# For node2
kubectl drain k8s-node2 --ignore-daemonsets --delete-emptydir-data
```

This will:
- Mark the node as unschedulable
- Evict all pods (except DaemonSets)
- Move workloads to other nodes

### 7.2 Delete the Node from Cluster (Run on Control Plane)

```bash
# For node1
kubectl delete node k8s-node1

# For node2
kubectl delete node k8s-node2
```

Verify the node is removed:

```bash
kubectl get nodes
```

### 7.3 Reset the Worker Node (Run on the Worker Node)

On the worker node you want to detach, run:

```bash
sudo kubeadm reset
```

You'll be prompted to confirm. Type `y` and press Enter.

### 7.4 Clean Up (Run on the Worker Node)

Remove residual configuration:

```bash
# Remove CNI configuration
sudo rm -rf /etc/cni/net.d

# Remove kubelet configuration
sudo rm -rf /var/lib/kubelet

# Remove etcd data (if any)
sudo rm -rf /var/lib/etcd

# Restart containerd
sudo systemctl restart containerd
```

### 7.5 Optional: Complete Cleanup (Run on the Worker Node)

If you want to completely remove Kubernetes from the node:

```bash
# Stop services
sudo systemctl stop kubelet

# Remove Kubernetes packages
sudo apt-mark unhold kubelet kubeadm kubectl
sudo apt-get purge -y kubelet kubeadm kubectl
sudo apt-get autoremove -y

# Remove configuration files
sudo rm -rf ~/.kube
sudo rm -rf /etc/kubernetes
```

### 7.6 Rejoin the Node (Optional)

If you want to rejoin the node later:

1. Ensure the node is properly configured (Part 1 steps)
2. Generate a new join command on the control plane:
   ```bash
   kubeadm token create --print-join-command
   ```
3. Run the join command on the worker node
4. If it's a GPU node, ensure NVIDIA Container Toolkit is configured

### 7.7 Emergency Node Removal

If a node is unresponsive and you can't drain it normally:

```bash
# Force delete the node (on control plane)
kubectl delete node k8s-node1 --force --grace-period=0

# Clean up any stuck pods
kubectl get pods -A -o wide | grep k8s-node1
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

---

## Summary

Your Kubernetes cluster is now ready with:
- 1 control plane node (10.20.21.57)
- 2 GPU-enabled worker nodes with NVIDIA L40 GPUs
- Flannel pod network
- NVIDIA device plugin for GPU scheduling

You can now deploy GPU-accelerated workloads to your cluster!

**Node Management:** You know how to safely detach and rejoin worker nodes as needed for maintenance or reconfiguration.

---

**About the Author**: Debashis Banerjee is an AI Solution Architect with 18+ years of experience in enterprise AI infrastructure and cloud-native solutions.

*Have questions or want to discuss GPU cluster optimization? Connect with me on [LinkedIn](https://linkedin.com/in/daverbj) or [Twitter](https://twitter.com/daverbj).*
