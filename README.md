# 🚀 k8s-2tier-production-app

A production-grade 2-Tier Kubernetes application built on a local KIND cluster. This project demonstrates real-world Kubernetes patterns used in production environments — including autoscaling, node scheduling, sidecar containers, and health monitoring.

---

## 🏗️ What I Built

This project runs a **2-Tier application** inside Kubernetes:

- **Frontend** — Nginx web server that serves the UI
- **Backend** — Python API that handles business logic

Both tiers run in separate Kubernetes pods, on separate nodes, with automatic scaling, health checks, and resource management — exactly how production apps run at companies like Swiggy, Razorpay, and Freshworks.

---

## 🖥️ Cluster Setup

The cluster runs locally using **KIND (Kubernetes IN Docker)** — which spins up a real Kubernetes cluster inside Docker containers on my laptop.

The cluster has **3 nodes**:
- 1 **Control Plane** — the brain of the cluster, manages everything
- 1 **Worker Node 1** — dedicated to running Frontend pods
- 1 **Worker Node 2** — dedicated to running Backend pods

Worker Node 1 and Worker Node 2 are completely separated — frontend pods never land on the backend node and vice versa. This separation is enforced using **Taints and Tolerations**.

---

## 🔴 Taints & Tolerations — Node Separation

**The problem:** By default, Kubernetes can schedule any pod on any node. In production, you don't want your frontend and backend sharing the same machine — a backend memory leak could crash the frontend too.

**The solution — Taints:**

Worker Node 1 has a taint `app=frontend:NoSchedule` — this is like putting a sign on the node saying *"Frontend pods only. All others stay out."*

Worker Node 2 has a taint `app=backend:NoSchedule` — same idea, backend pods only.

**Tolerations** are the pods' response to that sign. The Frontend deployment has a toleration that says *"I can handle the frontend taint — let me in."* The Backend deployment tolerates the backend taint. Any other pod without a matching toleration simply cannot be scheduled on these nodes.

**Real-world use case:** Companies taint GPU nodes so only AI/ML workloads run on expensive GPU machines. Normal pods can't accidentally land there and waste money.

---

## 🔵 Node Affinity — Preferred Scheduling

Taints tell nodes *who to reject*. Node Affinity tells pods *where they prefer to go*.

The Frontend deployment is configured with Node Affinity that says *"I prefer to run on Worker Node 1."* The Backend says *"I prefer Worker Node 2."*

This is a **soft rule** (preferred, not required) — if the preferred node is full or unavailable, the pod can still go elsewhere. This makes the system resilient while still keeping things organized under normal conditions.

**Combined effect of Taint + Affinity:** Frontend pods are attracted to Worker Node 1 by affinity AND can only enter because of their toleration. Backend pods do the same on Worker Node 2. The result is clean, predictable, separated deployments.

---

## 🟡 HPA — Horizontal Pod Autoscaler

**The problem:** Traffic is unpredictable. At 2 PM, 10 users visit. At 8 PM, 10,000 users visit. If you run only 2 pods always, you either waste money during off-peak hours or crash during peak hours.

**The solution — HPA:**

HPA watches CPU usage every 15 seconds using the metrics-server. When CPU crosses 70%, it automatically creates more pods. When traffic drops, it removes pods.

- **Frontend:** Starts with 2 pods. Scales up to 10 pods maximum when CPU > 70%.
- **Backend:** Starts with 2 pods. Scales up to 8 pods maximum when CPU > 70%.

The HPA never goes below 2 pods (minimum) — so there's always at least 2 pods ready even during quiet times.

**Real-world use case:** Zomato uses HPA during dinner hours (7-9 PM). Their pods scale from 5 to 50 automatically. After midnight, back to 5. This saves significant cloud costs while handling traffic spikes without manual intervention.

---

## 🟢 Health Probes — Readiness & Liveness

Every container in this project has two health checks configured.

**Readiness Probe** answers the question: *"Is this pod ready to receive traffic?"*

When a pod starts, it takes a few seconds to initialize. Without a readiness probe, Kubernetes might send user traffic to a pod that's still warming up — causing errors. The readiness probe checks if the pod is actually ready. Only when it passes does Kubernetes add the pod to the Service's load balancer and start sending it traffic.

**Liveness Probe** answers the question: *"Is this pod still alive and working?"*

Sometimes a process can be running but completely broken — stuck in a deadlock, out of memory, or frozen. The liveness probe detects this. If the probe fails 3 times in a row, Kubernetes automatically restarts the container. This is how Kubernetes achieves self-healing.

- **Frontend probes:** HTTP GET request to `/` on port 80 — if Nginx responds, it's healthy.
- **Backend probes:** TCP socket check on port 5000 — if the port is open, the API is alive.

---

## 🟣 Sidecar Pattern — Multi-Container Pods

Each pod in this project runs **two containers** — not one.

The main container runs the actual application (Nginx or Python API). The second container is a **sidecar** — a helper that runs alongside the main container and shares its network and storage.

In this project, the sidecar is a `log-collector` container built on BusyBox. It runs an infinite loop that prints a heartbeat log every 30 seconds. In a real production setup, this sidecar would collect logs and ship them to a centralized logging system like ELK or Loki — without changing the main application code at all.

**Why sidecar:** The main app doesn't need to know about logging. The sidecar handles it separately. This follows the single responsibility principle — each container does one thing.

---

## ⚪ Resource Requests & Limits

Every container in this project has explicit resource boundaries defined.

**Requests** are the *minimum guaranteed* resources. The Kubernetes scheduler uses this number to decide which node can fit the pod. If a node doesn't have at least this much free CPU and memory, the pod won't be scheduled there.

**Limits** are the *maximum allowed* resources. If a container tries to use more memory than its limit, Kubernetes kills it immediately (called OOMKilled — Out of Memory Killed). If it exceeds CPU limit, it gets throttled (slowed down, not killed).

Without limits, a single buggy pod could consume all node resources and starve every other pod on that node. With limits, each pod is contained in its own resource boundary regardless of what it does.

| Container | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| Frontend (Nginx) | 100m | 200m | 64Mi | 128Mi |
| Sidecar (log-collector) | 10m | 20m | 16Mi | 32Mi |
| Backend (Python API) | 100m | 300m | 128Mi | 256Mi |

*(100m = 100 millicores = 0.1 CPU core)*

---

## 🌐 Services — How Pods Talk to Each Other

**Frontend Service** is of type `NodePort`. It exposes the frontend application on port `30080` of the host machine. This means you can open a browser on your laptop and visit `localhost:30080` to see the Nginx welcome page. NodePort makes the service accessible from outside the cluster.

**Backend Service** is of type `ClusterIP`. It is only accessible from inside the cluster — no external access at all. The frontend pod can call the backend using its service name `backend-service` as the hostname. Kubernetes DNS resolves this to the correct pod automatically. No one from outside the cluster can reach the backend directly, which is the correct security posture.

---

## 📁 Project Structure

```
k8s-2tier-production-app/
├── README.md
├── kind-config.yaml          ← cluster definition (1 control-plane + 2 workers)
├── namespace.yaml            ← creates the "production" namespace
├── frontend/
│   ├── deployment.yaml       ← Nginx + Sidecar + Probes + Affinity + Toleration + Limits
│   ├── service.yaml          ← NodePort service on :30080
│   └── hpa.yaml              ← Autoscale 2→10 pods when CPU > 70%
└── backend/
    ├── deployment.yaml       ← Python API + Sidecar + Probes + Affinity + Toleration + Limits
    ├── service.yaml          ← ClusterIP service (internal only)
    └── hpa.yaml              ← Autoscale 2→8 pods when CPU > 70%
```

---

## 🚀 How to Run

```bash
# 1. Create the cluster
kind create cluster --name production --config kind-config.yaml

# 2. Add taints to worker nodes
kubectl taint nodes production-worker app=frontend:NoSchedule
kubectl taint nodes production-worker2 app=backend:NoSchedule

# 3. Install metrics-server (needed for HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 4. Deploy everything
kubectl apply -f namespace.yaml
kubectl apply -f frontend/
kubectl apply -f backend/

# 5. Verify
kubectl get all -n production
```

Open `http://localhost:30080` in your browser — you should see the Nginx welcome page.

---

## ✅ Verify Everything Works

```bash
# See all pods and which node they landed on
kubectl get pods -n production -o wide

# Watch HPA scaling in real time
kubectl get hpa -n production -w

# Check sidecar logs
kubectl logs -n production -l app=frontend -c log-collector

# Test self-healing — delete a pod and watch it come back
kubectl delete pod -n production -l app=frontend
kubectl get pods -n production -w
```

---

## 📚 Learning Reference

Built following **Cloud With VarJosh — CKA Certification Course 2025** (Day 1–24)

---

