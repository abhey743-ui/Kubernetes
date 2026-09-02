# Config Server Deployment + Service — Full Reference Guide

This document explains **your exact YAML file** (Deployment + Service for `configserver`) — what every keyword means, why it's written that way, and then walks through **everything that happens behind the scenes**, step by step, from the moment you run `kubectl apply` to a real user reaching your app through load balancing.

Use this as a **reference you can come back to** any time you write a similar Deployment + Service file.

---

## PART A — Understanding the YAML Itself (Reference)

Your file has **two objects in one file**, separated by `---`:
1. A **Deployment** (lines 1–17) → manages running your app
2. A **Service** (lines 19–29) → makes your app reachable over the network

### A.1 The Deployment Section

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configserver-deployment
  labels:
    app: configserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: configserver
  template:
    metadata:
      labels:
        app: configserver
    spec:
      containers:
        - name: configserver
          image: eazybytes/configserver:s14
          ports:
            - containerPort: 8071
```

| Keyword | What It Means | Why It's Important |
|---|---|---|
| `apiVersion: apps/v1` | Deployments belong to the `apps` API group, version 1 | Kubernetes needs this to know how to interpret the rest of the fields correctly |
| `kind: Deployment` | You're creating a **Deployment** object | Deployments are the standard way to run and manage replicated apps |
| `metadata.name` | The name of THIS Deployment object (`configserver-deployment`) | Used to reference/manage it later (`kubectl get deployment configserver-deployment`) |
| `metadata.labels` | Tags attached to the Deployment object itself | Mostly for organizing/searching — not critical to how it functions |
| `spec.replicas: 3` | **How many identical Pods to always keep running** | This is what gives you redundancy — if 1 crashes, Kubernetes replaces it to keep 3 alive |
| `spec.selector.matchLabels` | Tells the Deployment **which Pods belong to it** (must match `app: configserver`) | This is how the Deployment "claims ownership" of the Pods it creates |
| `spec.template` | The **blueprint** used to stamp out each of the 3 Pods | Everything under `template` describes what each Pod looks like |
| `template.metadata.labels` | The label stuck onto every Pod created from this template | **Must match `spec.selector.matchLabels` above**, or the Deployment won't recognize its own Pods (common beginner mistake) |
| `template.spec.containers` | List of containers to run inside each Pod | You have one container per Pod here |
| `containers[].name` | Name of the container (`configserver`) | Used for logs/debugging (`kubectl logs <pod> -c configserver`) |
| `containers[].image` | The Docker image to run (`eazybytes/configserver:s14`) | This is pulled from Docker Hub and used to start the container |
| `containers[].ports.containerPort` | The port the app listens on **inside** the container (`8071`) | This is informational/documentation for Kubernetes — it doesn't by itself expose anything externally (that's the Service's job) |

### A.2 The Service Section

```yaml
apiVersion: v1
kind: Service
metadata:
  name: configserver
spec:
  selector:
    app: configserver
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 8071
      targetPort: 8071
```

| Keyword | What It Means | Why It's Important |
|---|---|---|
| `apiVersion: v1` | Services are a **core** Kubernetes object | Uses the basic API version, unlike Deployment which needed `apps/v1` |
| `kind: Service` | You're creating a **Service** object | Services provide stable networking for a group of Pods |
| `metadata.name: configserver` | The name of this Service | Other apps inside the cluster can reach your Pods using this name as a hostname: `http://configserver:8071` |
| `spec.selector.app: configserver` | **Which Pods this Service sends traffic to** | Must match the label on your Pods (`app: configserver`) — this is the critical link between Service and Deployment (explained deeply in Part C) |
| `spec.type: LoadBalancer` | The type of Service — determines how it's exposed | `LoadBalancer` requests an external IP (in cloud environments) so the app is reachable from outside the cluster, not just internally |
| `spec.ports[].protocol: TCP` | The network protocol used | TCP is standard for HTTP-based apps like this |
| `spec.ports[].port: 8071` | The port **the Service itself** listens on | Other things reach your app using this port |
| `spec.ports[].targetPort: 8071` | The port **on the Pod/container** to forward traffic to | Must match `containerPort` in the Deployment |

### A.3 Quick Rule of Thumb When Writing Your Own
When writing a Deployment + Service pair from scratch, always make sure these 3 labels/ports line up correctly — this is the #1 source of bugs for beginners:

```
Deployment: spec.selector.matchLabels        ─┐
Deployment: template.metadata.labels          ├─ ALL THREE MUST MATCH
Service:    spec.selector                    ─┘

Deployment: containers[].ports.containerPort ─┐
Service:    ports[].targetPort               ─┴─ MUST MATCH
```

---

## PART B — What Happens When You Run `kubectl apply -f this-file.yaml` (Step by Step)

Let's trace the **entire journey**, from your terminal to a running, network-reachable app.

### Step 1: You Send the Request
```powershell
kubectl apply -f configserver.yaml
```
`kubectl` reads your file, converts both objects (Deployment and Service) into API requests, and sends them to the **API Server**.

### Step 2: The API Server Validates and Stores
- The API Server checks the YAML is valid (correct fields, correct types).
- It passes through internal **admission controllers** (automatic checks/defaults).
- It writes **both objects** (the Deployment and the Service) into **etcd** — this is the moment they officially "exist" in the cluster, even though nothing is running yet.

### Step 3: The Deployment Controller Takes Over
A component inside the **Controller Manager** called the **Deployment Controller** notices: *"a new Deployment exists, wanting `replicas: 3`, but 0 Pods currently match `app: configserver`."*

It creates a **ReplicaSet** — an intermediate object whose only job is to make sure exactly 3 Pods matching that label always exist.

### Step 4: The ReplicaSet Creates 3 Pod Objects
The **ReplicaSet Controller** creates **3 separate Pod objects** in etcd — each one an exact copy of your `template` block, each tagged with `app: configserver`. At this point, these Pods exist as **data/objects only** — still nothing is physically running yet.

### Step 5: The Scheduler Assigns Each Pod to a Node
The **Scheduler** notices 3 new Pods with no assigned node. For each Pod, it looks at all available Worker Nodes and picks the best one based on available CPU/memory and any constraints — then writes "this Pod belongs to Node X" back into etcd.

👉 **Important:** on your local Docker Desktop setup, there's only **1 Worker Node**, so all 3 Pods get scheduled onto that same single node. In a real multi-node cluster, the Scheduler would likely **spread these 3 Pods across different nodes** for better fault tolerance (explained more in Part D).

### Step 6: kubelet Starts the Actual Containers
On each assigned Worker Node, **kubelet** (the node's local agent) notices a Pod has been assigned to it. It tells the **Container Runtime (Docker)**:
1. Pull the image `eazybytes/configserver:s14` (if not already cached locally).
2. Start a container from that image.
3. Open port `8071` inside that container's network namespace.

This happens **independently and in parallel** for all 3 Pods (even if they're on the same node, each Pod gets its own isolated container + IP address inside the cluster's internal network).

### Step 7: Pods Report Back as "Running"
Once each container starts successfully, kubelet reports the Pod's status back to the API Server as `Running`. This is exactly what you'd see if you ran:
```powershell
kubectl get pods
```
```
NAME                                    READY   STATUS    RESTARTS   AGE
configserver-deployment-xxxxx-aaaa      1/1     Running   0          10s
configserver-deployment-xxxxx-bbbb      1/1     Running   0          10s
configserver-deployment-xxxxx-cccc      1/1     Running   0          10s
```

### Step 8: The Service Comes Alive (Networking Layer)
Separately (this can happen at the same time as Steps 3–7), the **Service Controller** and **kube-proxy** react to the Service object you created:
- The Service gets assigned a stable **internal cluster IP** (called a `ClusterIP` — this exists even for `LoadBalancer` type services, as the internal address).
- **kube-proxy** (running on every node) sets up networking rules (using iptables/IPVS internally) so that any traffic sent to the Service's IP/port gets automatically routed to one of the 3 healthy Pods.
- Because `type: LoadBalancer` was specified, Kubernetes also asks the underlying platform (a cloud provider in production, or Docker Desktop locally) to provision an **external-facing address**, so the app can be reached from outside the cluster too.

At this point — **your app is fully deployed, running in 3 places, and reachable** both internally (by other apps in the cluster) and externally (via the LoadBalancer).

---

## PART C — How the Deployment and Service Actually Interact (The Missing Link)

This is the part that confuses a lot of learners: **the Deployment and Service never directly reference each other by name.** There's no field in the Service saying "connect to Deployment `configserver-deployment`." Instead, they're connected **indirectly, through labels** — this is one of the most important design ideas in Kubernetes.

```
Deployment creates Pods with label:  app: configserver
                                            │
                                            │ (label matching — NOT a direct link)
                                            ▼
Service watches for Pods with label: app: configserver
```

The Service **continuously watches** the cluster for any Pod (from any source — Deployment, ReplicaSet, manually created, anything) carrying the label `app: configserver`, and automatically adds it as a valid traffic target. This means:
- If a Pod crashes and a replacement is created, the Service **automatically** picks it up — no manual reconfiguration needed.
- If you scale `replicas: 3` up to `replicas: 5`, the 2 new Pods are **automatically** included in the Service's traffic routing, purely because they share the same label.

👉 This label-based matching (not hard-coded names) is exactly why Kubernetes can self-heal and scale so smoothly — the Service never needs to "know" about individual Pods by name; it just trusts the label.

---

## PART D — Networking Deep Dive: Exposure & Load Balancing

### D.1 How the App Gets Exposed to the Outside World

Since you set `type: LoadBalancer`, here's the exposure chain:

```
Internet / Outside User
        │
        ▼
External Load Balancer IP  (provided by cloud provider, or simulated locally)
        │
        ▼
Kubernetes Service (configserver) — ClusterIP: internal stable address, Port: 8071
        │
        ▼
kube-proxy routing rules (spread traffic across matching Pods)
        │
   ┌────┼────┐
   ▼    ▼    ▼
 Pod1  Pod2  Pod3   (each running the configserver container on port 8071)
```

- On a **real cloud provider** (AWS/GCP/Azure), `type: LoadBalancer` automatically provisions a **real external load balancer** (like an AWS ELB) with a public IP.
- On **Docker Desktop locally**, there's no real cloud load balancer, so Docker Desktop simulates this by making the service reachable via `localhost` on the specified port — good enough for local testing.

### D.2 How Load Balancing Actually Works Internally

When a request hits the Service's address/port, **kube-proxy** decides which of the 3 Pods should handle it. By default, this uses a simple, efficient method (round-robin or random selection depending on the underlying mode — iptables or IPVS), spreading requests roughly evenly across all healthy Pods matching the label.

Importantly:
- kube-proxy only routes to Pods that are marked **healthy/ready** — if one Pod is still starting up or crashed, it's automatically excluded until it recovers.
- This load balancing happens **entirely inside the cluster's networking layer** — your app code doesn't need to do anything special to benefit from it.

### D.3 The Specific Endpoint Other Services Use to Reach Config Server

Inside the cluster, any other app (e.g., another microservice) can reach your Config Server using **Kubernetes' built-in internal DNS**, simply via the Service name:
```
http://configserver:8071
```
Kubernetes automatically creates a DNS entry for every Service, named exactly after `metadata.name` (in your case, `configserver`). This means other microservices never need to know Pod IPs, node names, or how many replicas exist — they just call this one stable address, and the Service + kube-proxy handle routing to whichever Pod is available.

---

## PART E — Worker Nodes: One Instance Per Node, or Many?

You asked a great question: **does a Worker Node run only one instance of a service, or can multiple instances live on the same node?**

**Answer: A single Worker Node CAN run multiple Pods of the same service at once** — there's no rule against it. What actually determines the spread is the **Scheduler's decision-making**, not a hard limit.

| Scenario | What Typically Happens |
|---|---|
| **Single-node cluster** (like your Docker Desktop setup) | All 3 replicas get scheduled onto the **same** node — there's simply no other option |
| **Multi-node cluster** (3+ Worker Nodes) | The Scheduler tries to **spread replicas across different nodes** where possible, for better fault tolerance — this is influenced by built-in scheduling preferences and can be made stricter using `podAntiAffinity` rules |
| **A node runs out of resources** | The Scheduler will avoid placing more Pods there, even if it means uneven distribution |

👉 In production clusters, engineers often explicitly configure **anti-affinity rules**, forcing Kubernetes to avoid putting all replicas of the same app on one node — this way, if a single physical/virtual machine goes down, you don't lose all 3 copies of your app at once. This wasn't specified in your YAML, so right now Kubernetes is just using its default best-effort spreading logic.

---

## PART F — Full Summary (Plain English Recap)

1. You wrote a Deployment that says: *"Always keep 3 copies of my Config Server app running."*
2. You wrote a Service that says: *"Give whatever Pods carry the `app: configserver` label one stable network address, and expose it externally via a LoadBalancer."*
3. When applied, the **API Server → Controller Manager → ReplicaSet → Scheduler → kubelet → Docker** chain actually creates and runs the 3 Pods.
4. The **Service** doesn't know about the Deployment directly — it just continuously watches for Pods matching its label selector, making the system self-healing and scale-friendly automatically.
5. **kube-proxy** handles real network traffic distribution (load balancing) across all matching, healthy Pods.
6. Other services inside the cluster reach your app simply via `http://configserver:8071` — Kubernetes' internal DNS handles the rest.
7. Whether all 3 replicas land on one node or spread across many depends entirely on how many Worker Nodes exist and the Scheduler's placement logic (or explicit anti-affinity rules, if you add them).

---

## PART G — Official Documentation (For Formal Reading)

- Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Pods: https://kubernetes.io/docs/concepts/workloads/pods/
- ReplicaSet: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/
- Services: https://kubernetes.io/docs/concepts/services-networking/service/
- Service Types (ClusterIP/NodePort/LoadBalancer): https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
- kube-proxy: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/
- kube-scheduler: https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/
- Labels and Selectors: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
- Pod Affinity/Anti-Affinity: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity
- DNS for Services: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

---

*Reference companion for your `configserver` Deployment + Service YAML — pairs with your other Kubernetes learning notes.*
