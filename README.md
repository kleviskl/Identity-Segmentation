# Identity Segmentation  
🐝 **Cilium Identity-Based Segmentation Demo**  

This project demonstrates using [Cilium](https://cilium.io) for identity-based network segmentation in Kubernetes.  
We’ll create a Kind cluster, deploy test workloads, and apply policies that allow traffic only between approved identities.  

# ---

## Overview  
1. Spin up a single-node `kind` cluster with **Cilium** as the CNI.  
2. Deploy three pods: `frontend`, `backend`, and `intruder`.  
3. Apply two `CiliumNetworkPolicy` resources:  
   - **default-deny-all-ingress** – blocks all inbound traffic.  
   - **allow-frontend-backend** – allows only traffic from pods with `role=frontend` to `role=backend`.  
4. Test connectivity to see Cilium enforce identity-based rules.  

# ---

## Prerequisites  
- Docker `27.5.1` (add your user to docker group)  
  ```bash
  sudo usermod -aG docker $USER && newgrp docker

	•	Kind 0.29.0
	•	kubectl v1.30+
	•	Cilium CLI 1.17.x or newer
	•	Tested on Ubuntu 24.04

# ⸻

Setup

1️⃣ Create project folder

mkdir -p ~/identityseg/manifest && cd ~/identityseg


# ⸻

2️⃣ Create kind-config.yaml

Save the following into manifest/kind-config.yaml:

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: id-seg-demo
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            system-reserved: cpu=200m,memory=512Mi
    extraPortMappings:
      - containerPort: 30000
        hostPort: 30000


# ⸻

3️⃣ Create Kind cluster

kind create cluster --name id-seg-demo --config manifest/kind-config.yaml
kubectl get nodes --watch


# ⸻

4️⃣ Install Cilium

cilium install --version 1.17.6
cilium status --wait


# ⸻

5️⃣ Deploy workloads

kubectl apply -f manifest/frontend.yaml \
              -f manifest/backend.yaml \
              -f manifest/backend-svc.yaml
kubectl get pods
kubectl get svc backend


# ⸻

6️⃣ Apply network policies

kubectl apply -f manifest/default-deny.yaml
kubectl apply -f manifest/allow-frontend-backend.yaml
kubectl get ciliumnetworkpolicies


# ⸻

Testing

Deploy intruder pod

kubectl run intruder --image=alpine --labels role=intruder \
  --command -- sleep 3600
kubectl wait --for=condition=ready pod/intruder --timeout=60s

Create test_seg.sh

#!/usr/bin/env bash
set -e
echo "✅ frontend -> backend (should pass)"
kubectl exec frontend -- wget -qO- backend | head -1

echo "✅ intruder -> backend (should fail)"
kubectl exec intruder -- wget -qO- --timeout=3 backend >/dev/null 2>&1 && \
  echo "❌ Unexpectedly succeeded" || echo "Blocked as expected"

Make executable:

chmod +x script/test_seg.sh

Run:

script/test_seg.sh

Expected output:

✅ frontend -> backend (should pass)
<!DOCTYPE html>
✅ intruder -> backend (should fail)
Blocked as expected


# ⸻

Optional: Inspect Cilium identities

CILIUM_POD=$(kubectl -n kube-system get pod -l k8s-app=cilium \
  -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system exec "$CILIUM_POD" -- cilium identity list

You should see something like:

k8s:role=backend
k8s:role=frontend
k8s:role=intruder


⸻

Monitoring Traffic

Watch dropped packets

kubectl -n kube-system exec -it "$CILIUM_POD" -- \
  cilium monitor --type drop

Watch all policy verdicts for backend

BACKEND_ID=$(kubectl get cep backend -o jsonpath='{.status.id}')
kubectl -n kube-system exec -it "$CILIUM_POD" -- \
  cilium monitor --type policy-verdict --related-to "$BACKEND_ID"

You’ll see allow verdicts for frontend and deny verdicts for intruder.

# ⸻

Troubleshooting: Cilium Pods Stuck in Pending

If you see something like:

DaemonSet cilium          Desired: 1, Unavailable: 1/1
Deployment cilium-operator Desired: 1, Unavailable: 1/1

and pods remain in Pending state, here are common causes and fixes:
	1.	Docker not running
	•	Kind requires Docker to be running before creating the cluster.
	•	Start Docker Desktop or run:

sudo systemctl start docker


	2.	Kind node lacks network permissions
	•	Restart the Kind cluster and reinstall Cilium with minimal dependencies:

kind delete cluster --name id-seg-demo
kind create cluster --name id-seg-demo --config manifest/kind-config.yaml
cilium install \
  --version 1.15.5 \
  --set routingMode=tunnel \
  --set kubeProxyReplacement=disabled \
  --set hostServices.enabled=false \
  --set externalIPs.enabled=true \
  --set nodePort.enabled=true \
  --set hubble.enabled=false


	3.	CRDs missing for CiliumNetworkPolicy
	•	If you get errors like no matches for kind "CiliumNetworkPolicy", it means Cilium CRDs are not yet installed.
	•	Re-run:

cilium install
cilium status --wait


	4.	Cluster resource shortage
	•	Increase Kind node resources in kind-config.yaml:

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            system-reserved: cpu=200m,memory=512Mi
    extraPortMappings:
      - containerPort: 30000
        hostPort: 30000



After fixing, check again:

kubectl -n kube-system get pods -l k8s-app=cilium
kubectl -n kube-system get pods -l io.cilium/app=operator


# ⸻

1. Spin up a single‑node `kind` cluster with Cilium as the CNI.  
2. Deploy three pods (`frontend`, `backend`, `intruder`).  
3. Apply two CiliumNetworkPolicies:  
   * `default‑deny‑all‑ingress` – blocks all.  
   * `allow‑frontend‑backend`  – re‑opens traffic from `role=frontend` to `role=backend`.  
4. Generate traffic and watch Cilium **allow** or **deny** packets based on the pods’ labels.

### Prerequisites:
- Note: commands were run on Ubuntu 24
- install:
    - Docker: version 27.5.1
      - sudo usermod -aG docker $USER && newgrp docker
    - Kind: version 0.29.0
    - kubectl: v1.30+
    - Cilium: latest 1.17.x

## Setup:
- Make project directory:
  ```
  mkdir -p ~/identityseg/manifests && cd ~/identityseg
  ```
  
- Create kind-config.yaml (notes if you are not grabbing GitHub files).

- Create a cluster:
```
kind create cluster --name id-seg-demo --config manifest/kind-config.yaml
```
```
kubectl get nodes --watch
```

- Install Cilium:
```
cilium install --version 1.17.6
cilium status --wait
```

- Create frontend.yaml, backend.yaml, and backend-svc.yaml (notes if you are not grabbing GitHub files)

- Apply workloads:
```
kubectl apply -f manifest/frontend.yaml \
             -f manifest/backend.yaml \
             -f manifest/backend-svc.yaml
```

- Sanity check:
```
kubectl get pods
kubectl get svc backend
```

- Create default-deny.yaml and allow-frontend-backend.yaml (notes if you are not grabbing GitHub files).

- Apply workloads:
```
kubectl apply -f manifest/default-deny.yaml

kubectl apply -f manifest/allow-frontend-backend.yaml
```

- Sanity check:
```
kubectl get ciliumnetworkpolicies
kubectl get ciliumnetworkpolicy default-deny-all-ingress -o wide
```

## Steps:

#### In first terminal:

- Deploy intruder pod:
```
kubectl run intruder --image=alpine --labels role=intruder \
  --command -- sleep 3600
kubectl wait --for=condition=ready pod/intruder --timeout=60s
```

- Create test_seg.sh script and make script executable:

- Write test_seg.sh:
```
#!/usr/bin/env bash
set -e
echo "✅ frontend -> backend (should pass)"; echo
kubectl exec frontend -- wget -qO- backend | head -1
echo
echo "✅ intruder -> backend (should fail)"; echo
kubectl exec intruder -- wget -qO- --timeout=3 backend >/dev/null 2>&1 && \
  echo "❌  Unexpectedly succeeded" || echo "Blocked as expected"
```

- Make executable:
```
chmod +x script/test_seg.sh
```

- In first terminal, run script:
```
script/test_seg.sh
```

Expected test_seg.sh script run output:
```
~/identityseg$ script/test_seg.sh
✅ frontend -> backend (should pass)

<!DOCTYPE html>

✅ intruder -> backend (should fail)

Blocked
```

- Optional: Confirm pod identities. The Cilium policies are tied to these identities:

Run the following command to see the roles confirmed:
```
kubectl -n kube-system exec "$CILIUM_POD" -- cilium identity list | \
```
Output should look similar to:
```
k8s:role=backend
k8s:role=frontend
k8s:role=intruder
```
Run the following command to list the Cilium identities:
```
kubectl -n kube-system exec "$CILIUM_POD" -- cilium identity list
```
Output should look similar to:
```
8608    k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default
        k8s:io.cilium.k8s.policy.cluster=kind-id-seg-cluster
        k8s:io.cilium.k8s.policy.serviceaccount=default
        k8s:io.kubernetes.pod.namespace=default
        k8s:role=backend
    9532    k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default
        k8s:io.cilium.k8s.policy.cluster=kind-id-seg-cluster
        k8s:io.cilium.k8s.policy.serviceaccount=default
        k8s:io.kubernetes.pod.namespace=default
        k8s:role=frontend
    50744   k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default
        k8s:io.cilium.k8s.policy.cluster=kind-id-seg-cluster
        k8s:io.cilium.k8s.policy.serviceaccount=default
        k8s:io.kubernetes.pod.namespace=default
        k8s:role=intruder
```

#### In second terminal, run:

```
CILIUM_POD=$(kubectl -n kube-system get pod -l k8s-app=cilium \
              -o jsonpath='{.items[0].metadata.name}')
```

- Optional: check to confirm agent grabbed policy
```
kubectl -n kube-system exec "$CILIUM_POD" -- cilium policy get | less
```

- Get the pod:
```
kubectl -n kube-system exec -it "$CILIUM_POD" -- \
  cilium endpoint list | grep backend
```

- Monitor for traffic to demonstrate policy in action:
```
kubectl -n kube-system exec -it "$CILIUM_POD" -- cilium monitor --type drop
```

- Rerun test_seg.sh script in first terminal.

- Check output in second terminal with expected output similar to:
```
xx drop (Policykubectl get ciliumnetworkpolicy default-deny-all-ingress -o wide
 denied) flow 0x7c2352cd to endpoint 354, ifindex 9, file bpf_lxc.c:2118, , identity 50744->8608: 10.xxx.0.xx:52140 -> 10.xxx.0.xxx:80 tcp SYN
```

#### Monitor All Traffic
 - The previous monitor command only catches traffic that is dropped, but when the following
 commands are run, both approved and denied/dropped traffic is visible.

 - Get the endpoint ID for the pod:
```
 BACKEND_ID=$(kubectl get cep backend -o jsonpath='{.status.id}')
echo "Backend endpoint ID = $BACKEND_ID"
```

- If needed, set CILIUM_POD again:
```
CILIUM_POD=$(kubectl -n kube-system get pod -l k8s-app=cilium \
              -o jsonpath='{.items[0].metadata.name}')
```

- Use the altered monitor command to monitor traffic:
```
 kubectl -n kube-system exec -it "$CILIUM_POD" -- \
  cilium monitor --type policy-verdict --related-to "$BACKEND_ID"
```

- Expected output should show both accepted and dropped traffic. While traffic from the frontend to the backend should pass, traffic from the intruder to the backend should fail.
- Run test_seg.sh script again in the first terminal. Expected output of monitor command should look similar to:
```
Policy verdict log: flow 0x100a8441 local EP ID 354, remote ID 9532, proto 6, ingress, action allow, auth: disabled, match L3-Only, 10.xxx.0.xxx:38178 -> 10.xxx.0.xxx:80 tcp SYN
Policy verdict log: flow 0x83556614 local EP ID 354, remote ID 50744, proto 6, ingress, action deny, auth: disabled, match none, 10.xxx.0.xx:34098 -> 10.xxx.0.xxx:80 tcp SYN
Policy verdict log: flow 0xb2b031f9 local EP ID 354, remote ID 50744, proto 6, ingress, action deny, auth: disabled, match none, 10.xxx.0.xx:34098 -> 10.xxx.0.xxx:80 tcp SYN
Policy verdict log: flow 0xf50c1462 local EP ID 354, remote ID 50744, proto 6, ingress, action deny, auth: disabled, match none, 10.xxx.0.xx:34098 -> 10.xxx.0.xxx:80 tcp SYN
Policy verdict log: flow 0xae4eea63 local EP ID 354, remote ID 9532, proto 6, ingress, action allow, auth: disabled, match L3-Only, 10.xxx.0.xxx:44950 -> 10.xxx.0.xxx:80 tcp SYN
```

Author: Klevis Koleci