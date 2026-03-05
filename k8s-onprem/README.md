# Juice Shop — On-Premise Kubernetes Deployment

Deploy OWASP Juice Shop on a bare-metal / on-premise Kubernetes cluster using:

| Component | Role |
|-----------|------|
| **Calico** | CNI — pod networking and `NetworkPolicy` enforcement |
| **MetalLB** | Load-balancer — assigns LAN IPs to `LoadBalancer` Services |
| **Longhorn** | Block storage (optional) — distributed PVC storage across nodes |

> **Architecture note:** Juice Shop is deployed as a **stateless** workload. The application
> resets on restart by design, so no persistent volume is required. Traffic is served directly
> via a MetalLB `LoadBalancer` Service — no ingress controller needed.

---

## Prerequisites

### 1. Calico
Install Calico as the cluster CNI (skip if already installed):
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/calico.yaml
```
Verify:
```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
```

### 2. MetalLB
Install MetalLB in L2 mode:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

# Wait for MetalLB controller to be ready
kubectl rollout status deployment/controller -n metallb --timeout=90s
```
Then apply the address pool. **Edit the IP range first** to match a free range on your LAN:
```bash
# Open prerequisites/metallb-config.yaml and set spec.addresses to your LAN range
kubectl apply -f prerequisites/metallb-config.yaml
```
Verify:
```bash
kubectl get ipaddresspool -n metallb
kubectl get l2advertisement -n metallb
```

> **MetalLB namespace:** This setup uses the `metallb` namespace (not `metallb-system`).
> Ensure port **7946 TCP/UDP** is open on all nodes — MetalLB speakers use this for memberlist
> gossip. Without it, ARP for the VIP will be unreliable and the external IP will be unreachable.

### 3. Longhorn (optional)
Longhorn is not required for this deployment but is available if you want persistent storage
for other workloads on the cluster:
```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/deploy/longhorn.yaml
```
> Requires `open-iscsi` on every node: `yum install iscsi-initiator-utils` (RHEL/CentOS) or `apt install open-iscsi` (Debian/Ubuntu)

---

## Deployment

### Step 1 — Configure MetalLB IP range
Edit `prerequisites/metallb-config.yaml` and set `spec.addresses` to a free IP range on your LAN:
```yaml
spec:
  addresses:
    - 192.168.1.200-192.168.1.220   # replace with your range
```
Apply it:
```bash
kubectl apply -f prerequisites/metallb-config.yaml
```

### Step 2 — Deploy Juice Shop
Apply the full stack with Kustomize:
```bash
kubectl apply -k deploy/
```

Or apply files individually:
```bash
kubectl apply -f deploy/namespace.yaml
kubectl apply -f deploy/deployment.yaml
kubectl apply -f deploy/service.yaml
kubectl apply -f deploy/hpa.yaml
kubectl apply -f deploy/networkpolicy.yaml
```

### Step 3 — Verify
```bash
# Check all resources in the namespace
kubectl get all -n juice-shop

# Confirm pods are Running and 1/1 Ready
kubectl get pods -n juice-shop

# Confirm MetalLB has assigned an EXTERNAL-IP
kubectl get svc juice-shop -n juice-shop
```
Expected output:
```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
juice-shop   LoadBalancer   10.x.x.x        192.168.1.200    80:3XXXX/TCP   1m
```

### Step 4 — Access
Open a browser and navigate to:
```
http://<EXTERNAL-IP>
```
Or verify from the command line:
```bash
EXTERNAL_IP=$(kubectl get svc juice-shop -n juice-shop -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s -o /dev/null -w "%{http_code}" http://$EXTERNAL_IP/rest/admin/application-version
# Expected: 200
```

---

## File Overview

### `deploy/` — Application manifests (applied via `kubectl apply -k deploy/`)

| File | Description |
|------|-------------|
| `deploy/namespace.yaml` | `juice-shop` namespace with Calico networking label |
| `deploy/deployment.yaml` | Deployment — 2 replicas, topology spread across nodes, no persistent storage |
| `deploy/service.yaml` | `LoadBalancer` Service — MetalLB assigns an external IP from the configured pool |
| `deploy/hpa.yaml` | HPA — scales 2–10 replicas at 70% CPU / 80% memory utilization |
| `deploy/networkpolicy.yaml` | Calico `NetworkPolicy` — allow inbound on port 3000, allow DNS + HTTP/HTTPS egress |
| `deploy/kustomization.yaml` | Kustomize entry-point |

### `prerequisites/` — Cluster-level config (applied once, outside Kustomize)

| File | Description |
|------|-------------|
| `prerequisites/metallb-config.yaml` | MetalLB `IPAddressPool` + `L2Advertisement` — set your LAN IP range here |

---

## Scaling

The HPA automatically scales pods between **2 and 10 replicas**:
```bash
# View current HPA status
kubectl get hpa -n juice-shop

# Manually set replicas (overridden by HPA after stabilization window)
kubectl scale deployment/juice-shop -n juice-shop --replicas=4
```

---

## Updating the Image

To roll out a new image version:
```bash
kubectl set image deployment/juice-shop juice-shop=bkimminich/juice-shop:<new-tag> -n juice-shop
kubectl rollout status deployment/juice-shop -n juice-shop
```
To roll back:
```bash
kubectl rollout undo deployment/juice-shop -n juice-shop
```

---

## Troubleshooting

### EXTERNAL-IP stays `<pending>`
1. Check MetalLB controller is running:
   ```bash
   kubectl get pods -n metallb
   ```
2. Verify the address pool name matches the annotation in `deploy/service.yaml`:
   ```bash
   kubectl get ipaddresspool -n metallb
   # service annotation: metallb.universe.tf/address-pool: default
   ```
3. Confirm port 7946 TCP/UDP is open between all nodes (MetalLB memberlist).

### External IP is assigned but browser times out
Check the `NetworkPolicy` — traffic from outside the cluster arrives with a node IP as source.
The ingress rule must use `ipBlock: cidr: 0.0.0.0/0`, not `namespaceSelector`, to allow it:
```bash
kubectl describe networkpolicy juice-shop-network-policy -n juice-shop
```

### Pod is in `CrashLoopBackOff`
```bash
kubectl logs -n juice-shop -l app=juice-shop --previous
kubectl describe pod -n juice-shop -l app=juice-shop
```
Common cause: a `PersistentVolumeClaim` mounted at `/juice-shop/data` will shadow the
image-baked static files and cause `ENOENT` errors. The PVC is disabled by default for this reason.

### Rolling update is stuck
If `maxUnavailable: 0` causes a stall (old pod holds a `ReadWriteOnce` PVC), the deployment
uses `maxUnavailable: 1` to ensure the old pod is terminated before the new one starts:
```bash
kubectl rollout status deployment/juice-shop -n juice-shop
kubectl rollout restart deployment/juice-shop -n juice-shop   # force a fresh rollout
```
