# Istio Internal Architecture — Deep Dive (Istio-Specific Mechanics Only)

This assumes you're already comfortable with Docker, containers, pods, and kubectl/Kubernetes fundamentals. We're skipping all of that and going straight into what's specific to Istio — and every Istio-specific term or mechanism gets fully explained exactly when it shows up, no skipping.

---

## 1. Sidecar Injection — The Exact Mechanism

"Injection" is Istio's term for adding the Envoy proxy container into your pod automatically. Here's exactly how, mechanically:

### 1.1 The Mutating Admission Webhook — what it exactly does
Kubernetes has a built-in extension point called an **admission webhook**: any time a pod is about to be created, Kubernetes can be configured to call out to an external service and ask "any objections, or changes you want to make, before I actually create this?" There are two kinds:
- A **validating** webhook can only approve/reject.
- A **mutating** webhook can actually rewrite the object before it's created.

Istio registers a **mutating webhook** (part of `istiod`) for this exact purpose. When you label a namespace `istio-injection=enabled`, you're telling Kubernetes: "for any pod created here, call Istio's webhook and let it modify the pod spec."

The webhook's exact job: take the incoming pod spec (as JSON) and return a **JSON patch** — a set of precise add/modify instructions — that adds:
- The `istio-init` container (explained in 1.2)
- The `istio-proxy` container (the actual Envoy sidecar)
- Extra volumes for certificates and config that these containers need
- Annotations/labels used for tracking

This all happens in milliseconds, before the pod is ever scheduled to a node.

### 1.2 What `istio-init` Actually Configures (iptables specifics)
`istio-init` is an **init container** — Kubernetes guarantees init containers run to completion before any regular container in the pod starts. Its exact job is to run `iptables` commands (using elevated `NET_ADMIN` capability) that set up **traffic interception rules** inside that pod's network namespace specifically. In concrete terms, it sets rules that:

- Redirect all **outbound** TCP traffic from the app container to `localhost:15001` — the port Envoy listens on for outbound traffic.
- Redirect all **inbound** traffic destined for the app's ports to `localhost:15006` — the port Envoy listens on for inbound traffic.
- Exclude Envoy's own traffic and specific ports (like the health check port) from redirection, so you don't get infinite redirect loops.

Note: in newer Istio versions, this can alternatively be done by a **CNI plugin** (`istio-cni`) instead of a per-pod init container — same effect (traffic redirection rules), but configured once at the node level instead of inside every single pod. Either way, the *result* is identical: transparent redirection, zero app-level configuration.

### 1.3 Why Two Separate Ports (15001 vs 15006)?
This distinction matters because Envoy needs to treat outbound and inbound traffic with different logic:
- **15001 (outbound)**: Envoy acts as a client-side proxy — it decides *where* to send the request (load balancing, retries, circuit breaking logic all apply here).
- **15006 (inbound)**: Envoy acts as a server-side proxy — it enforces *who's allowed in* (mTLS verification, authorization policy checks apply here).

Same Envoy binary, two logically separate roles depending on traffic direction — this is why people describe each pod's Envoy as handling both "client-side" and "server-side" proxying simultaneously.

---

## 2. Istiod's Exact Internal Responsibilities

`istiod` is a single binary, but it's the merger of three historically separate Istio components. Knowing the old names helps because a lot of docs/diagrams still reference them:

| Old component name | What it exactly did (now folded into istiod) |
|---|---|
| **Pilot** | Converts your `VirtualService`/`DestinationRule`/`Gateway` objects into Envoy-native config and pushes it via xDS |
| **Citadel** | Runs the internal Certificate Authority (CA) — issues short-lived X.509 certificates to each workload identity |
| **Galley** | Validates and processes configuration input before it's distributed (ensures malformed YAML doesn't get pushed out and break the mesh) |

You'll still see these three names in architecture diagrams and older documentation even though they're now one process.

### 2.1 Workload Identity — exactly what gets certified
Each workload (roughly: each unique service account in Kubernetes) gets an identity encoded using the **SPIFFE** standard (Secure Production Identity Framework For Everyone) — an industry-standard format for workload identities, structured like:

```
spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>
```

This exact string is embedded inside the X.509 certificate Citadel/istiod issues to that workload. When two Envoys do an mTLS handshake, they're not just checking "is this certificate validly signed" — they're checking this SPIFFE identity string against `AuthorizationPolicy` rules, which is what lets you write rules like "only workloads with service account `checkout` may call this service," rather than relying on IP addresses (which change constantly in Kubernetes).

### 2.2 Certificate Rotation
These certificates are deliberately **short-lived** (default: 24 hours, configurable). istiod automatically rotates them before expiry, silently, over the same connection — this limits the damage if a certificate/key were ever compromised, since it becomes useless within hours regardless.

---

## 3. The xDS Protocol — Exact Mechanics

"xDS" collectively refers to Envoy's **discovery service APIs** — the specific gRPC services Envoy calls to fetch its configuration, rather than reading it from a static file. Each Envoy holds one persistent **ADS (Aggregated Discovery Service)** gRPC stream open to istiod — "aggregated" meaning all the individual xDS types below are multiplexed over that single connection rather than needing separate connections each.

Exact breakdown of what each sub-API delivers, and the exact order Envoy typically needs them in to avoid misrouting traffic during updates:

1. **CDS (Cluster Discovery Service)** — delivers **clusters**: in Envoy's terminology, a "cluster" is a named group of upstream endpoints that share a load-balancing policy (e.g., "the `payment-service-v1` cluster"). This is the first thing Envoy needs, since routes/endpoints reference clusters by name.
2. **EDS (Endpoint Discovery Service)** — delivers the **actual current IP:port list** of healthy pods backing each cluster. This updates continuously as pods scale up/down or fail health checks.
3. **LDS (Listener Discovery Service)** — delivers **listeners**: the exact ports/protocols Envoy should bind to and which filter chains (processing logic) to apply to traffic hitting each one.
4. **RDS (Route Discovery Service)** — delivers **routes**: the compiled-down form of your `VirtualService` rules — literally "if request matches X, send to cluster Y with Z% weight."
5. **SDS (Secret Discovery Service)** — delivers the **actual TLS certificate/private key material**, fetched dynamically rather than mounted as a static file, so rotation (2.2) can happen without restarting Envoy.

Istiod pushes updates to these incrementally and only to the Envoys that actually need them (this scoping is called **config distribution scoping**) — a mesh with thousands of services doesn't push every Envoy the entire mesh's config, only what's relevant to what that specific workload can legally talk to (based on Kubernetes namespace/`Sidecar` resource scoping).

---

## 4. Traffic Policy Evaluation — Exact Order of Operations

When `order-service`'s outbound Envoy (port 15001, per Section 1.3) processes a call to `payment-service`, it evaluates things in this exact order:

1. **Listener match** — which filter chain applies, based on destination port/protocol.
2. **Route match (RDS)** — matches the request against `VirtualService` rules (by URI path, headers, etc.) to pick a destination — could be a specific subset/version.
3. **Cluster selection (CDS/EDS)** — resolves that destination to its cluster definition and current healthy endpoint list.
4. **Load balancing policy** — applies the algorithm defined in `DestinationRule` (`ROUND_ROBIN`, `LEAST_CONN`, `RANDOM`, or `CONSISTENT_HASH` for session affinity) to pick one specific endpoint from that list.
5. **Outlier detection check** — Envoy tracks recent success/failure rates per endpoint; if an endpoint has been failing (per thresholds you configure), it's temporarily **ejected** from the load-balancing pool — this is the exact mechanism behind "circuit breaking."
6. **Connection pool limits** — checks against configured max connections/requests to avoid overwhelming a single endpoint.
7. **Retry/timeout policy** — if the call fails and a retry policy is defined (`VirtualService.retries`), Envoy automatically retries against a *different* endpoint (not the one that just failed), up to the configured attempt count, respecting the overall timeout budget.

On the receiving Envoy (port 15006), the exact order is:
1. **mTLS termination** — decrypt the connection, extract the peer's SPIFFE identity from its certificate.
2. **PeerAuthentication check** — was mTLS actually required for this workload, and was it satisfied? (`PeerAuthentication` can be set to `STRICT`, `PERMISSIVE`, or `DISABLE` per namespace/workload.)
3. **AuthorizationPolicy evaluation** — check the caller's identity/attributes against any `ALLOW`/`DENY` rules; default is allow-all unless a policy exists, but once *any* `ALLOW` policy exists for a workload, it becomes implicit-deny for everything not explicitly listed.
4. **Local rate limiting** (if configured) — reject/queue if request rate exceeds a defined threshold.
5. **Hand off to the application container** over `localhost`.

---

## 5. Observability — Exact Data Path

Nothing here is guessed — Envoy generates three exact categories of telemetry natively, with no app instrumentation:

- **Metrics**: Envoy exposes a `/stats/prometheus` endpoint on each sidecar with counters/histograms for every request (`istio_requests_total`, `istio_request_duration_milliseconds`, tagged by source/destination workload, response code, etc.). A Prometheus server scrapes these on an interval.
- **Access logs**: each Envoy can be configured to emit a structured log line per request (configurable format) to stdout, collected by whatever log pipeline you run (e.g., Fluentd → Elasticsearch).
- **Distributed tracing**: Envoy propagates and augments trace headers (`x-request-id`, B3 headers, or W3C Trace Context) on every hop, and reports span data to a tracing backend (Jaeger/Zipkin/Tempo) — this is *why* tracing works across services with zero app code changes, as long as your app forwards the incoming trace headers onto any calls it makes downstream (this one part — header propagation — is the single thing your app code does need to cooperate with).

---

## 6. Ambient Mode — Exact Architectural Difference

Since you already know the sidecar internals above, here's precisely what's different in Ambient Mode:

- **ztunnel** replaces the per-pod Envoy for L4 (network-layer) duties only: it runs as one process per node (a Kubernetes **DaemonSet** — a workload type guaranteed to run exactly one copy per node), and handles mTLS origination/termination and identity for every pod on that node, using the same certificate/SPIFFE mechanism from Section 2.
- Traffic redirection to ztunnel happens via a **CNI plugin** at the node level (not per-pod iptables/init containers) — functionally the same redirection concept as Section 1.2, just implemented once per node instead of once per pod.
- **Waypoint proxies** are actual Envoy instances (same binary as sidecar mode) but deployed as their own separate pods (not injected into your app pod), one per service or namespace, that traffic gets routed through *only when* an L7 (HTTP-level) rule exists for that destination — i.e., a `VirtualService` with path-based routing, or an `AuthorizationPolicy` referencing HTTP-level attributes like headers.
- Everything from Section 3 (xDS) and Section 4 (policy evaluation order) still applies identically — it's the same Envoy engine and the same config protocol, just relocated out of your app's pod and split across ztunnel (L4) and waypoints (L7).

---

## 7. Precise Term Reference (Istio-Specific Only)

| Term | Exact meaning |
|---|---|
| Mutating admission webhook | Kubernetes extension point letting Istio rewrite a pod spec before creation |
| istio-init | Init container that sets up iptables traffic-redirection rules per pod |
| istio-cni | Node-level alternative to istio-init; does the same redirection via a CNI plugin instead |
| istio-proxy | The actual Envoy container name inside an injected pod |
| Cluster (Envoy term) | A named group of upstream endpoints sharing a load-balancing config |
| Listener (Envoy term) | A bound port + processing logic (filter chain) for traffic hitting it |
| Filter chain | The sequence of processing steps Envoy applies to traffic on a listener |
| ADS | Aggregated Discovery Service — the single gRPC stream multiplexing all xDS types |
| SPIFFE identity | Standardized workload identity string embedded in each certificate |
| Outlier detection | Envoy's mechanism for temporarily ejecting a failing endpoint from load balancing (= circuit breaking) |
| PeerAuthentication | Config object controlling whether/how mTLS is enforced for a workload |
| AuthorizationPolicy | Config object controlling which identities may call a workload |
| DaemonSet | Kubernetes workload type that runs exactly one pod copy per node (used for ztunnel) |
| ztunnel | Ambient mode's per-node proxy handling L4 mTLS/identity |
| Waypoint proxy | Ambient mode's optional per-service/namespace Envoy for L7 rules |

---

## 8. What To Inspect, Exactly, To Verify Each Section

- `kubectl describe pod <name>` — confirms the exact containers from Section 1 (`istio-init`, `istio-proxy`) and their exact command/args.
- `istioctl proxy-config cluster <pod>` — dumps the exact CDS data (Section 3.1) that pod's Envoy currently holds.
- `istioctl proxy-config route <pod>` — dumps the exact RDS data (compiled `VirtualService` rules).
- `istioctl proxy-config endpoint <pod>` — dumps exact EDS data (live healthy IPs).
- `kubectl exec <pod> -c istio-proxy -- curl localhost:15000/stats/prometheus` — pulls the raw metrics described in Section 5, directly from that pod's Envoy admin interface (port 15000).
