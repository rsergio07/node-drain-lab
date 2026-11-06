# node-drain-lab

Drain a Kubernetes node and watch customers fail, then use a PodDisruptionBudget, topology spread and a preStop hook to do the same work with far less impact.

**Video walkthrough:** on the [Reliability Bench](https://www.youtube.com/@reliabilitybench) YouTube channel

## What you will learn

- What kubectl drain does
- Why replicas alone do not protect you
- How PodDisruptionBudgets work
- How to spread pods and shut down gracefully

**SLO:** 99% of checkouts finish in under 500ms.

## Requirements

Docker with 4 CPUs and 8 GiB of memory, plus `git`, `make`, `kind`, `kubectl`, `helm`, `jq` and `curl`. Open source only, and it runs locally. The first run downloads several images.

Tested on one setup: macOS arm64, Colima 0.10.3 (4 CPUs, 8 GiB, `vz`), kind 0.33.0 with Kubernetes 1.37, Helm 4.3.1 and kube-prometheus-stack 92.18.0. On Colima, first run:

```bash
colima ssh -- sudo sysctl -w fs.inotify.max_user_instances=1024 fs.inotify.max_user_watches=1048576
```

## The lab

Clone the repository, then run every command from its root.

```bash
git clone https://github.com/rsergio07/node-drain-lab.git node-drain-lab
cd node-drain-lab
```

### Step 1: Check your machine

Every line should say `ok`.

```bash
make preflight
```

### Step 2: Create the cluster

The private `.kubeconfig` keeps this lab away from your other clusters. Run the `export` again in any new terminal.

```bash
kind create cluster --config cluster/kind.yaml --kubeconfig .kubeconfig
export KUBECONFIG=$PWD/.kubeconfig
kubectl get nodes
```

### Step 3: Build the app and load it into the cluster

kind nodes have their own container runtime, so the image must be loaded into them.

```bash
docker build -t shop:lab app
kind load docker-image shop:lab --name lab
```

### Step 4: Install monitoring

This installs Prometheus and Grafana with only what the lab needs.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack --version 92.18.0 -n monitoring --create-namespace -f deploy/monitoring-values.yaml --wait --timeout 8m
```

### Step 5: Deploy the shop on one node

Cordoning worker2 first puts every storefront pod on one node, which is what can happen without a spread rule.

```bash
kubectl cordon lab-worker2
kubectl apply -f deploy/dashboard.yaml
kubectl apply -f deploy/shop.yaml
kubectl -n shop rollout status deployment/storefront --timeout=2m
kubectl uncordon lab-worker2
kubectl -n shop get pods -o wide
```

### Step 6: Measure the baseline

Wait for traffic to build up.

```bash
# Wait 90 seconds
./scripts/verdict.sh 60
```

### Step 7: Drain the node

Three replicas, all on the node you are draining.

```bash
kubectl drain lab-worker --ignore-daemonsets --delete-emptydir-data
# Wait 45 seconds
./scripts/verdict.sh 60
kubectl -n shop get pods -o wide -l app=storefront
```

### Step 8: Bring the node back

Pods do not move back on their own.

```bash
kubectl uncordon lab-worker
kubectl get nodes
```

### Step 9: Protect the storefront

The budget keeps 2 pods available. The spread rule separates them. The preStop hook lets traffic drain first.

```bash
cat deploy/pdb.yaml
kubectl apply -f deploy/pdb.yaml
kubectl apply -f deploy/storefront-resilient.yaml
kubectl -n shop rollout status deployment/storefront --timeout=3m
# Wait 75 seconds
kubectl -n shop get pods -o wide -l app=storefront
kubectl -n shop get pdb
```

### Step 10: Drain the node again

Watch the evictions happen one by one.

```bash
kubectl drain lab-worker --ignore-daemonsets --delete-emptydir-data
# Wait 45 seconds
./scripts/verdict.sh 60
kubectl -n shop get pods -o wide -l app=storefront
```

### Step 11: Clean up

```bash
kind delete cluster --name lab --kubeconfig .kubeconfig
```

## Challenges

Predict the result before each run.

1. Set `minAvailable: 3`. What does the drain do?
2. Remove the preStop hook and compare the lost customers.
3. Drain both app nodes at once.

## Safety

The lab never changes your default Kubernetes context. Grafana allows anonymous viewing and its ports are for local use. Do not expose them.

## License

MIT
