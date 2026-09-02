# Kubernetes Dashboard — Internal Architecture & Who Does What (Behind the Scenes)

This document explains the **internal, technical working** of everything that happened during your Dashboard setup — not just the commands, but **which Kubernetes component was actually responsible** for creating, running, and connecting each piece. Think of this as "what really happened backstage" when you ran each command.

---

## 1. The Big Picture: 4 Layers Involved

Everything you did touches **4 different layers** of the system, each with a different "responsible party":

```
Layer 1: Your Machine (Client Layer)
   → kubectl, Helm, PowerShell

Layer 2: Control Plane (Decision-Making Layer)
   → API Server, etcd, Scheduler, Controller Manager

Layer 3: Worker Node (Execution Layer)
   → kubelet, Container Runtime (Docker), the actual Dashboard containers

Layer 4: Networking Layer
   → kube-proxy, Kong Gateway (Dashboard's built-in proxy), port-forward tunnel
```

Every command you ran passed through some (or all) of these layers. Let's trace each one.

---

## 2. What ACTUALLY Happens When You Run `helm install`

This is the most misunderstood part — Helm feels like magic, but it's not.

### Step-by-step internal flow:

1. **Helm (on your machine)** reads the **Chart** you pulled from the repo (`kubernetes-dashboard/kubernetes-dashboard`). A Chart is just a folder full of **YAML templates** — Deployment, Service, ServiceAccount, ConfigMap, RBAC rules, etc. — with placeholders for values.

2. **Helm renders the templates** — it fills in the placeholders using default (or your custom) `values.yaml`, producing **final, real Kubernetes YAML** in memory. (Nothing is created yet — this is just text generation, happening entirely on your machine.)

3. **Helm sends this rendered YAML to the Kubernetes API Server** — exactly the same way `kubectl apply -f` would, just automated for many files at once. **Helm itself has no special power inside the cluster** — it's just a client, like `kubectl`, that happens to be smart about templating and bundling.

4. **The API Server validates and stores everything in `etcd`** — every object (Deployment, Service, ServiceAccount, etc.) that Helm generated gets written into the cluster's database.

5. **Helm also creates a hidden tracking object** (a Secret, by default) recording "Release: kubernetes-dashboard, Revision 1, these are the resources I installed" — this is how `helm list`, `helm upgrade`, and `helm uninstall` later know exactly what to change or remove.

6. From here on, **Helm's job is done**. It doesn't run anything, doesn't manage anything ongoing — the actual running of the Dashboard is now entirely the Control Plane and Worker Node's responsibility (Sections 3–4 below).

### 👉 Who is "responsible" for the install?
**Helm** = translator/packager (client-side only). **The API Server + etcd** = the actual source of truth once installed.

---

## 3. Who Actually Creates and Runs the Dashboard Pods

After Helm hands off the rendered YAML to the API Server, here's who takes over:

| Component | Responsibility |
|---|---|
| **API Server** | Receives the Deployment object, validates it, stores it in etcd. Does NOT create the actual container. |
| **Controller Manager** (specifically the **Deployment Controller** + **ReplicaSet Controller** inside it) | Notices a new Deployment exists with "desired replicas: 1 (or more)." Creates a matching **ReplicaSet**, which then creates the actual **Pod** object. |
| **Scheduler** | Notices a new Pod exists with no assigned Node yet. Picks the best available Worker Node (in your case, just the single Docker Desktop node) and assigns the Pod to it. |
| **kubelet** (running on that Worker Node) | Watches the API Server, sees a Pod has been assigned to its node, and takes responsibility for actually starting it. |
| **Container Runtime (Docker)** | kubelet instructs Docker to pull the Dashboard's container images and start them as running containers. |

So when you saw `kubectl get pods -n kubernetes-dashboard` show `Running`, that status was the end result of **5 different components** each doing their specific job in sequence — Helm wasn't involved in any of this part at all.

### What's Actually Running Inside Those Pods?
The Dashboard chart typically creates **multiple containers/pods** working together (exact names can vary slightly by chart version):
- `kubernetes-dashboard-web` — serves the actual browser UI (HTML/JS/CSS).
- `kubernetes-dashboard-api` — the backend that talks to the Kubernetes API Server on your behalf and returns cluster data to the UI.
- `kubernetes-dashboard-auth` — handles login/token verification.
- `kubernetes-dashboard-kong-*` — an internal **API gateway** (Kong) that sits in front of all the above, so you connect to ONE address (`kong-proxy`) instead of juggling multiple internal services. This is exactly why your port-forward command targets `svc/kubernetes-dashboard-kong-proxy`.

---

## 4. Who Created the `admin-user` ServiceAccount — Internally

When you ran:
```powershell
kubectl apply -f dashboard-adminuser.yaml
```

Here's exactly what happened internally:

1. **kubectl** (your local client) read the YAML file and converted it into an API request — essentially an HTTP `POST`/`PATCH` request to the Kubernetes API Server.
2. **The API Server** received this request, checked it was syntactically valid (correct `apiVersion`, `kind`, required fields).
3. **Admission Controllers** (a set of internal checks inside the API Server) ran — these are automatic gatekeepers that can validate, mutate, or reject the object before it's saved (for a simple ServiceAccount, this step mostly just confirms it's allowed).
4. **The API Server wrote the object into `etcd`** — this is the actual moment the ServiceAccount "exists" in the cluster (a database row/entry, essentially).
5. A built-in Kubernetes component called the **ServiceAccount Controller** (part of the Controller Manager) noticed the new ServiceAccount and did some automatic bookkeeping (in modern Kubernetes, it no longer auto-generates a long-lived token by default — this is the exact reason you had to manually create the token via Secret or `kubectl create token`, as covered in your other notes file).

### 👉 Who is "responsible" for `admin-user` existing?
**You** decided it should exist (via YAML) → **kubectl** sent the request → **the API Server** validated and stored it → **etcd** is where it physically "lives" as data.

---

## 5. Who Enforces the ClusterRoleBinding — Internally (This Is the Interesting Part)

This is where a lot of people get confused — the ClusterRoleBinding doesn't "run" anything. It's **pure data** that gets **checked at request-time**, every single time.

### Here's what really happens every time the Dashboard (using `admin-user`'s token) tries to do something, e.g., "list all Pods":

1. Your browser (via the Dashboard's backend) sends a request to the **API Server**, including the `admin-user` token in the `Authorization` header.
2. **Authentication step**: The API Server checks — "is this token valid, and which identity does it belong to?" It confirms this token belongs to ServiceAccount `admin-user` in namespace `kubernetes-dashboard`.
3. **Authorization step (this is where RBAC lives)**: The API Server's built-in **RBAC Authorizer** module looks up: *"Is there any RoleBinding or ClusterRoleBinding connecting this identity to a Role/ClusterRole that permits this specific action (list Pods)?"*
4. It finds your `ClusterRoleBinding` object (stored in etcd, created back in Section 4's process) linking `admin-user` → `cluster-admin` ClusterRole.
5. It checks what `cluster-admin` allows — which is **literally everything** (`*` verbs on `*` resources in `*` API groups — the built-in role is intentionally unrestricted).
6. Since the action is allowed, the API Server proceeds and returns the Pod list.

### 👉 Who is "responsible" for enforcing this?
**The API Server itself**, specifically its internal **RBAC Authorization module** — this check happens on **every single API request**, not just once at setup. The ClusterRoleBinding you created isn't a one-time action — it's a **permanent rule** that gets consulted continuously, for as long as it exists.

---

## 6. Who Issues the Actual Token — Internally

### Method A: `kubectl create token admin-user`
This command talks to a specific, modern Kubernetes API called the **TokenRequest API**. Internally:
1. `kubectl` sends a request to the API Server: "generate a short-lived token for ServiceAccount `admin-user`."
2. The API Server's built-in **Service Account Token Issuer** (a component that signs tokens using the cluster's private signing key) creates a cryptographically signed **JWT (JSON Web Token)**.
3. This token is returned directly — it is **not stored anywhere** in etcd as a separate object. It's generated on-demand and simply expires after its lifetime (default ~1 hour).

### Method B: The Secret-based token (`secret.yaml`)
1. You created a Secret with the special annotation `kubernetes.io/service-account.name: "admin-user"`.
2. A controller called the **Legacy Service Account Token Controller** (still present for backward compatibility) noticed this annotation, generated a long-lived signed JWT the same way, and **wrote it into the Secret's data field** (base64-encoded) — permanently, until you delete the Secret.

### 👉 Who is "responsible" for the token?
Both methods ultimately rely on the same underlying cryptographic signer inside the **API Server / Controller Manager** — the difference is just **whether it's generated on-demand (Method A)** or **generated once and stored permanently (Method B)**.

---

## 7. Who Makes the Browser Actually Reach the Dashboard — Internally (Port-Forward Explained)

This is the final piece — how `https://localhost:8443` in your browser actually reaches a Pod running inside Docker Desktop's internal Kubernetes VM/containers.

### Internal flow of `kubectl port-forward`:

1. **kubectl** (your machine) opens a connection to the **API Server**.
2. The API Server has a special built-in **proxy subresource** — it doesn't forward traffic itself, but it authenticates you and then hands off the connection.
3. The API Server opens a **streaming connection (SPDY/WebSocket protocol) to the kubelet** on the node where the target Pod/Service is running.
4. **kubelet** then forwards that connection directly into the target container's network namespace — essentially opening a private tunnel straight into the Pod.
5. Kong (the Dashboard's internal gateway, running as one of the Pod's containers) receives the HTTPS request and routes it internally to whichever Dashboard component (`web`, `api`, `auth`) should handle it.
6. The response travels back through the exact same tunnel in reverse: **Pod → kubelet → API Server → kubectl → your browser.**

### 👉 Who is "responsible" for this connection?
It's a **relay chain**: `kubectl` (initiator) → `API Server` (authenticator + relay point) → `kubelet` (final delivery to the container) → `Kong` (internal app routing). **No new network resource (like a Service or Ingress) was permanently created for this** — it's a temporary, on-demand tunnel that exists only while your `port-forward` command keeps running, which is exactly why closing that terminal window kills your Dashboard access.

---

## 8. Full End-to-End Responsibility Table

| Step | What You Did | Who's Actually Responsible Internally |
|---|---|---|
| Install Dashboard | `helm install ...` | Helm (renders YAML) → API Server (validates+stores) → Controller Manager (creates ReplicaSet/Pods) → Scheduler (assigns node) → kubelet (starts containers) → Docker (runs them) |
| Create identity | `kubectl apply -f dashboard-adminuser.yaml` | kubectl (sends request) → API Server (validates+stores in etcd) |
| Grant permissions | `kubectl apply -f dashboard-rolebinding.yaml` | kubectl (sends request) → API Server (stores binding) → RBAC Authorizer (enforces it on every future request) |
| Get token | `kubectl create token` / Secret method | API Server's Token Issuer (signs JWTs using cluster's private key) |
| Access UI | `port-forward` + browser | kubectl → API Server (proxy/auth) → kubelet (tunnel to pod) → Kong (routes to right Dashboard container) |

---

## 9. One Unified Diagram (Everything Combined)

```
 YOU (PowerShell)
    │
    ├── helm install ──────────────► API Server ──► etcd (stores Deployment, Service, SA, RBAC objects)
    │                                    │
    │                                    ▼
    │                          Controller Manager
    │                        (creates ReplicaSet → Pod)
    │                                    │
    │                                    ▼
    │                              Scheduler
    │                        (assigns Pod to Worker Node)
    │                                    │
    │                                    ▼
    │                               kubelet
    │                        (tells Docker to run containers)
    │                                    │
    │                                    ▼
    │                        Dashboard Pods Running
    │                    (web / api / auth / kong-proxy containers)
    │
    ├── kubectl apply (ServiceAccount) ──► API Server ──► etcd (admin-user identity stored)
    │
    ├── kubectl apply (ClusterRoleBinding) ──► API Server ──► etcd (permission rule stored)
    │                                              │
    │                                     (checked on EVERY future request by RBAC Authorizer)
    │
    ├── kubectl create token ──► API Server's Token Issuer ──► signed JWT returned
    │
    └── kubectl port-forward ──► API Server (proxy) ──► kubelet ──► Pod (Kong → Dashboard UI)
                                                                          │
                                                                          ▼
                                                              YOUR BROWSER (https://localhost:8443)
```

---

## 10. Key Theoretical Takeaways

- **Nothing you did was "instant magic."** Every command triggered a chain of specific, named Kubernetes components, each with ONE clear job.
- **Helm is only a client-side templating tool** — once installation finishes, Helm has zero ongoing involvement; the Control Plane fully owns the running system from then on.
- **RBAC isn't a one-time check** — the ClusterRoleBinding you created is consulted by the API Server's RBAC Authorizer on literally every single request the Dashboard makes, forever (or until you delete/change it).
- **Tokens are cryptographically signed, not just random strings** — they're JWTs signed by the cluster's private key, which is why the API Server can trust and verify them without a database lookup every time (for the TokenRequest-based ones).
- **`kubectl port-forward` doesn't create any permanent infrastructure** — it's a live, temporary relay that only exists while the command is running; this is different from a Service or Ingress, which are permanent, always-on objects.
- **Every component has exactly one job** — this separation of responsibility (API Server only stores/validates, Scheduler only decides placement, kubelet only executes, RBAC Authorizer only checks permissions) is precisely what makes Kubernetes reliable, debuggable, and predictable at scale.

---

*Theoretical/internal-architecture companion — pairs with `kubernetes-dashboard-complete-setup.md` and `kubernetes-yaml-rbac-deep-dive.md`.*
