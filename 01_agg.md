# 01 - Aggregated Serving

Here, we will deploy NVIDIA Dynamo with aggregated serving on
a local Minikube cluster. This assumes you have completed all the steps in the [README.md](./README.md) file. 

---


### Step 1: Deploy Your First Model with Aggregated Serving

```bash
# Apply the deployment
kubectl apply -f serve/agg_router.yaml --namespace dynamo-system
```

Wait for it to become ready:

```bash
kubectl wait \
  --for=condition=Ready \
  dgd/vllm-agg-router \
  -n dynamo-system \
  --timeout=30m
```

Check the deployment status:

```bash
kubectl get dynamographdeployment -n dynamo-system
```

#### Step 2a: Verify the Dynamo Deployment Has Metrics Labels

```bash
kubectl get pods -l nvidia.com/metrics-enabled=true --show-labels -n dynamo-system
```

#### Step 2b: Check Logs

```bash
kubectl logs -n dynamo-system \
  -l nvidia.com/dynamo-graph-deployment-name=vllm-agg-router \
  --all-containers=true --prefix=true --timestamps -f
```

#### Step 2c: Get Dynamo Metrics

```bash
# Get the frontend pod name
FRONTEND_POD=$(kubectl get pods -n dynamo-system | grep frontend | head -1 | awk '{print $1}')
kubectl exec $FRONTEND_POD -n dynamo-system -- curl -s localhost:8000/metrics | head -20
```

---

### Step 3: Test the Deployment

```bash
export DGD_NAME=$(kubectl get dgd/vllm-agg-router \
  -n dynamo-system \
  --output=jsonpath='{.metadata.name}')
export FRONTEND_SERVICE="${DGD_NAME}-frontend"

# Forward the service port (run in background)
kubectl port-forward \
  --address 0.0.0.0 \
  --namespace dynamo-system \
  "service/${FRONTEND_SERVICE}" 8000:8000 \
  >/tmp/dynamo-port-forward.log 2>&1 &
export PORT_FORWARD_PID=$!
```

> Later, stop the port-forward with `kill "$PORT_FORWARD_PID"`.

Health and model checks:

```bash
curl --silent --fail http://localhost:8000/health
curl http://localhost:8000/v1/models
```

Non-streaming chat completion:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen2.5-1.5B-Instruct","messages":[{"role":"user","content":"Hello! How are you?"}],"stream":false,"max_tokens":50}'
```

Streaming chat completion:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen2.5-1.5B-Instruct","messages":[{"role":"user","content":"Write a short poem about AI"}],"stream":true,"max_tokens":100}'
```

Chat completion with sampling parameters:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen2.5-1.5B-Instruct","messages":[{"role":"user","content":"Explain quantum computing in one sentence"}],"stream":false,"temperature":0.7,"max_tokens":100,"top_p":0.9}'
```

#### Useful Metrics to Query

1. **Total requests to frontend:**
   ```
   dynamo_frontend_requests_total
   ```

2. **Time to first token (95th percentile):**
   ```
   histogram_quantile(0.95, dynamo_frontend_time_to_first_token_seconds_bucket)
   ```

3. **Request rate (per second):**
   ```
   rate(dynamo_frontend_requests_total[1m])
   ```

4. **Inter-token latency:**
   ```
   dynamo_frontend_inter_token_latency_seconds
   ```

---

### Step 4: Benchmarking with AI-Perf

```bash
uv pip install aiperf -q
```

#### Step 4a: Baseline Benchmark (Low Concurrency)

```bash
aiperf profile \
  --model Qwen/Qwen2.5-1.5B-Instruct \
  --url http://localhost:8000 \
  --endpoint-type chat \
  --streaming \
  --concurrency 1 \
  --request-count 100
```

#### Step 4b: Baseline Benchmark (High Concurrency)

```bash
# Test with higher concurrency to stress test
aiperf profile \
  --model Qwen/Qwen2.5-1.5B-Instruct \
  --url http://localhost:8000 \
  --endpoint-type chat \
  --streaming \
  --concurrency 4 \
  --request-count 200
```

#### Step 4c: Benchmark with Request Rate

```bash
# Test with a request rate instead of fixed concurrency
aiperf profile \
  --model Qwen/Qwen2.5-1.5B-Instruct \
  --url http://localhost:8000 \
  --endpoint-type chat \
  --streaming \
  --request-rate 10 \
  --request-count 200
```

---

### Step 5: Set Up Alerts

```bash
kubectl apply -f others/high-latency-alert.yaml -n monitoring
```

---

### Step 6: Autoscaling (Optional)

Uncomment the line `scalingAdapter: {}` in your deployment. You can then see a
DGDSA (Dynamo Graph Deployment Scaling Adapter) object:

```bash
kubectl get dgdsa -A
```

---

### Step 7: Cleanup

```bash
# Delete the deployment
kubectl delete dynamographdeployment vllm-agg-router -n dynamo-system
```
