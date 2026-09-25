# NVIDIA Dynamo  on Minikube

A step-by-step walkthrough for deploying NVIDIA Dynamo on
a local Minikube cluster, including full observability (Prometheus, Grafana, Loki,
Alloy) and benchmarking.

Reference: <https://docs.nvidia.com/dynamo/dev/kubernetes/getting-started/quickstart>

---

## Part 0: Minikube Setup

Start Minikube with GPU support (if configured):

```bash
minikube start \
  --driver docker \
  --container-runtime docker \
  --gpus all \
  --memory=16000mb \
  --cpus=8
```

Enable the required addons:

```bash
minikube addons enable istio-provisioner
minikube addons enable istio
minikube addons enable storage-provisioner-rancher
minikube addons enable metrics-server   # needed for autoscaling, etc.
```

Verify the setup:

```bash
# Check Minikube status
minikube status

# Verify Istio installation
kubectl get pods -n istio-system

# Verify storage class
kubectl get storageclass
```

### Troubleshooting: "too many open files"

If you hit `failed to create fsnotify watcher: too many open files`, raise the
open-file limits:

```bash
sudo sed -i '/^# End of file$/i *                hard    nofile          97816\n*                soft    nofile          97816' /etc/security/limits.conf
sudo sed -i '/^# End of file$/i '"$USER"' soft nofile 97816\n'"$USER"' hard nofile 97816' /etc/security/limits.conf
```

---

## Part 1: Setup and Deployment

### Step 1: Set Configuration Variables

```bash
export RELEASE_VERSION='1.5.0'
export HF_TOKEN='<your-huggingface-token>'
```


### Step 2: Verify Kubernetes Access

```bash
# Verify kubectl is installed and configured
kubectl version --client

# Check cluster connection
kubectl cluster-info

# Check GPU nodes are available (optional)
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPUs:.status.capacity.nvidia\\.com/gpu
```

### Step 3: Create Your Personal Namespace

```bash
# Create your personal namespace
kubectl create namespace dynamo-system

# Verify the namespace was created
kubectl get namespace
```

---

### Step 4: Observability


Since observability is a key component, we will enable it. If Prometheus is not
set up, metric enablement during the Dynamo installation will fail. You can
disable metrics, run Dynamo, and skip or postpone observability if needed.


#### Step 4a: Install Prometheus/Grafana via Helm

Add the Prometheus community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install `kube-prometheus-stack` in the `monitoring` namespace:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set-json 'prometheus.prometheusSpec.podMonitorNamespaceSelector={}' \
  --set-json 'prometheus.prometheusSpec.probeNamespaceSelector={}' \
  --set grafana.adminPassword=admin
```

Verify the monitoring stack pods:

```bash
kubectl get pods -n monitoring | grep -E "(prometheus|grafana|alertmanager|operator)"
```

#### Step 4b: Download and Apply Dynamo/Loki Dashboard ConfigMaps

You can download them from [Github repo](github.com/ai-dynamo/dynamo/raw/refs/tags/v1.5.0/deploy/observability/)

Apply the dashboards:

```bash
# Application dashboard
kubectl apply -n monitoring -f grafana-dynamo-dashboard-configmap.yaml

# Operator dashboard
kubectl apply -n monitoring -f grafana-operator-dashboard-configmap.yaml

# Confirm the ConfigMaps exist
kubectl get configmap -n monitoring | grep dynamo-dashboard
```

#### Step 4c: Set Up Loki

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana-community/loki \
  -n monitoring \
  -f loki-values.yaml
```


Optionally port-forward the Loki gateway (not required for the rest of the flow):

```bash
kubectl port-forward \
  --address 0.0.0.0 \
  --namespace monitoring \
  "service/loki-gateway" 3100:80 \
  >/tmp/loki-port-forward.log 2>&1 &
export LOKI_PORT_FORWARD_PID=$!
```

#### Step 4d: Deploy Alloy (Log Collector)

Alloy is what actually collects the logs and ships them to Loki.

```bash
sed -i 's/\$MONITORING_NAMESPACE/monitoring/g' alloy-values.yaml
sed -i 's/\$DYN_NAMESPACE/dynamo-system/g' alloy-values.yaml

# The k8s-monitoring chart lives in the grafana repo; add it if you haven't already
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install --values alloy-values.yaml alloy grafana/k8s-monitoring \
  --version 3.8.13 \
  -n monitoring
```

**Alternative: Deploy Alloy via manifests (only if above fails)**

If the Helm version does not work, apply the manifests directly:

```bash
# kubectl apply -f alloy-manifest-opt/
```

#### Step 4e: Configure Loki So It Listens to Alloy

```bash
# Logging dashboard and Loki data source
sed -i 's/\$MONITORING_NAMESPACE/monitoring/g' loki-datasource.yaml
kubectl apply -n monitoring -f loki-datasource.yaml
kubectl apply -n monitoring -f logging-dashboard.yaml
```

#### Step 4f: Restart Grafana to Load the Dashboard

```bash
kubectl delete pod -l app.kubernetes.io/name=grafana -n monitoring
```

#### Step 4g: Forward Prometheus and Grafana Ports (run in background)

```bash
kubectl port-forward \
  --address 0.0.0.0 \
  --namespace monitoring \
  "service/prometheus-kube-prometheus-prometheus" 9090:9090 \
  >/tmp/prometheus-port-forward.log 2>&1 &
export PROMETHEUS_PORT_FORWARD_PID=$!

kubectl port-forward \
  --address 0.0.0.0 \
  --namespace monitoring \
  "service/prometheus-grafana" 3000:80 \
  >/tmp/grafana-port-forward.log 2>&1 &
export GRAFANA_PORT_FORWARD_PID=$!
```

Test the Prometheus API:

```bash
curl -s http://localhost:9090/api/v1/query?query=up | head -20
```

Then navigate to <http://localhost:3000> to see the Grafana dashboards.



---

### Step 5: Install the Namespace-Scoped Dynamo Platform

This installs ETCD, NATS, and the Dynamo Operator Controller in your namespace
with namespace restriction enabled.

```bash
# Download the platform chart
helm fetch https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform-$RELEASE_VERSION.tgz

helm install dynamo-platform dynamo-platform-$RELEASE_VERSION.tgz \
  --namespace dynamo-system \
  --create-namespace \
  --set dynamo-operator.metricsService.enabled=true \
  --set "dynamo-operator.dynamo.metrics.prometheusEndpoint=http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090"

# Wait for the platform pods to be ready
kubectl wait --for=condition=ready pod \
  --all \
  --namespace dynamo-system \
  --timeout=300s
```

Verify the CRDs and pods:

```bash
kubectl get crd | grep nvidia.com
kubectl get pods -n dynamo-system
```

#### Step 5a: Check if PodMonitors Were Created

```bash
# Lists PodMonitors in your namespace (shows even if no workload is running)
kubectl get podmonitor -n dynamo-system
```

---

### Step 6: Create the HuggingFace Token Secret

```bash
# Create the HuggingFace token secret
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="$HF_TOKEN" \
  --namespace dynamo-system

# Verify the secret was created
kubectl get secret hf-token-secret -n dynamo-system
```

