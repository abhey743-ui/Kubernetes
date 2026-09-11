# Implementing mTLS for Service-to-Service Communication in Kubernetes (Istio)

This is a hands-on implementation guide — actual YAML, actual commands, in the actual order you'd run them. It assumes Istio is already installed in your cluster and your services are already sidecar-injected (from the previous guides). Every Istio-specific object gets explained exactly when it's used.

---

## 1. What You're Actually Turning On

Two separate config objects control mTLS, and you need to understand the split before writing any YAML:

- **`PeerAuthentication`** — controls the **server side**: "when traffic arrives at this workload, should it require mTLS, allow either, or reject it entirely?"
- **`DestinationRule`** — controls the **client side**: "when this workload sends traffic to a destination, should it originate an mTLS connection?"

Istio actually auto-manages the `DestinationRule` side for you in most cases (`istioctl` and Istiod cooperate to default clients to mTLS automatically once the mesh has certificates flowing), but you'll still write explicit `DestinationRule`s for anything you want to lock down or verify precisely. We'll do both explicitly below so nothing is left implicit.

---

## 2. Step 1 — Confirm Certificates Are Actually Being Issued

Before enabling any policy, verify the underlying certificate machinery (Section 2 of the internals deep-dive — istiod's CA function) is actually working:

```bash
istioctl proxy-config secret <pod-name> -n <namespace>
```

You should see a `default` resource with a `STATUS: Valid (Cert Chain)` and an expiry roughly 24 hours out. If this is empty or errors, mTLS cannot function — fix injection/istiod health first, since everything below assumes certificates already exist.

---

## 3. Step 2 — Start in `PERMISSIVE` Mode (Not `STRICT`)

`PeerAuthentication` supports three exact modes:

| Mode | Exact behavior |
|---|---|
| `STRICT` | Only accepts mTLS connections; plaintext is rejected outright |
| `PERMISSIVE` | Accepts **both** mTLS and plaintext on the same port, auto-detecting which one arrived |
| `DISABLE` | Requires plaintext only; mTLS is not accepted |

**Always start with `PERMISSIVE`.** The reason: it lets already-meshed services start using mTLS immediately while anything *not yet* sidecar-injected (a legacy pod, a health-check probe hitting the pod directly, a service outside the mesh) keeps working over plaintext during the transition. Jumping straight to `STRICT` on a live system is the single most common way to cause an outage during mTLS rollout — some caller you forgot about (often a Kubernetes liveness/readiness probe, which doesn't go through the mesh) gets rejected instantly.

### Apply mesh-wide `PERMISSIVE` mode:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: PERMISSIVE
```

Applying this in the `istio-system` namespace specifically makes it the **mesh-wide default** — every namespace inherits it unless overridden by a more specific `PeerAuthentication` (namespace or workload-level ones always take precedence over the mesh-wide one).

```bash
kubectl apply -f mesh-permissive-mtls.yaml
```

---

## 4. Step 3 — Verify Traffic Is Actually Encrypted

Don't trust the config — check the real traffic. From inside one pod's Envoy, inspect an actual connection to another:

```bash
istioctl proxy-config secret <destination-pod> -n <namespace>
```

More concretely, use the built-in mTLS check:

```bash
istioctl x describe pod <pod-name> -n <namespace>
```

This exact command reports, per workload, whether inbound traffic is mTLS, plaintext, or a mix — this is your ground truth, not the YAML you applied.

You can also directly capture and inspect traffic to confirm encryption:

```bash
kubectl exec <pod-name> -c istio-proxy -- tcpdump -i eth0 -A -s0 'port 8080' -c 5
```

If mTLS is active, this will show unreadable encrypted bytes on the wire rather than plaintext HTTP — that's your actual proof, not just "the policy exists."

---

## 5. Step 4 — Narrow the Scope As You Confirm Each Namespace

Rather than flipping the entire mesh to `STRICT` at once, migrate namespace by namespace, confirming each one with Step 3 before moving to the next:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-app-namespace
spec:
  mtls:
    mode: STRICT
```

This namespace-scoped object **overrides** the mesh-wide `PERMISSIVE` default from Step 2, for workloads in `my-app-namespace` only. Repeat this per namespace as you validate each one, rather than doing it globally in one shot.

### For a single workload, narrow it further with a selector:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: payment-service-strict
  namespace: my-app-namespace
spec:
  selector:
    matchLabels:
      app: payment-service
  mtls:
    mode: STRICT
  portLevelMtls:
    8080:
      mode: PERMISSIVE
```

That `portLevelMtls` block is the exact mechanism for a common real case: enforce `STRICT` mTLS mesh-traffic-wide for this workload, but keep one specific port (say, a health-check port hit directly by kubelet, bypassing the mesh) on `PERMISSIVE` so probes don't break.

---

## 6. Step 5 — Explicitly Set Client-Side `DestinationRule`

While Istio auto-configures client mTLS in most setups, write this explicitly for anything you want guaranteed and auditable — particularly before you flip a namespace to `STRICT`:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: payment-service-mtls
  namespace: my-app-namespace
spec:
  host: payment-service.my-app-namespace.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

`ISTIO_MUTUAL` is the exact value meaning "use Istio's own auto-managed certificates for mTLS" (as opposed to `SIMPLE`, which is plain one-directional TLS using certs you supply manually, or `MUTUAL`, which is mTLS using certs you supply manually rather than Istio's built-in CA). For internal service mesh mTLS, `ISTIO_MUTUAL` is what you want essentially every time.

**Important exact rule to know**: a `mesh`-wide `DestinationRule` (`host: "*.local"`, applied in `istio-system`) is a common pattern to set `ISTIO_MUTUAL` as the default for literally every internal call at once, rather than writing one per service:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: default-mtls
  namespace: istio-system
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

---

## 7. Step 6 — Final Cutover to Mesh-Wide `STRICT`

Once every namespace has been validated individually (Step 4/5) and nothing is still relying on plaintext, update the mesh-wide default itself:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

At this point you can also delete the namespace-level `STRICT` overrides from Step 4 — they're now redundant since the mesh-wide default already matches, though leaving them in place is harmless (an override with the same value as the default just does nothing extra).

---

## 8. Common Break Points, Exactly What Causes Them

| Symptom | Exact cause | Exact fix |
|---|---|---|
| Kubernetes readiness/liveness probes start failing after `STRICT` | Kubelet hits the pod's IP directly, bypassing the mesh — this traffic is plaintext and gets rejected under `STRICT` | Istio auto-rewrites probe traffic to hit Envoy first via the `status.sidecar.istio.io/inject` probe rewrite mechanism — confirm it's active, or use `portLevelMtls: PERMISSIVE` on the probe port |
| A specific external/legacy service can't reach a meshed service anymore | The caller has no sidecar, so it can't originate mTLS, and the destination is now `STRICT` | Either bring that caller into the mesh (inject a sidecar), or explicitly scope that one destination's `PeerAuthentication` to `PERMISSIVE` |
| Traffic looks fine in logs but `istioctl x describe` shows plaintext | Client-side `DestinationRule` is missing or set to a non-`ISTIO_MUTUAL` mode | Apply the explicit `DestinationRule` from Section 6 |
| mTLS works within a namespace but breaks cross-namespace | A namespace-scoped `PeerAuthentication` in the destination namespace conflicts with what the source namespace's `DestinationRule` is sending | Check both sides — mode mismatches between the two are the single most common cross-namespace mTLS bug |

---

## 9. Verifying the End State, Precisely

Final checklist, each with its exact command:

```bash
# 1. Confirm mesh-wide policy
kubectl get peerauthentication default -n istio-system -o yaml

# 2. Confirm no unexpected PERMISSIVE overrides remain
kubectl get peerauthentication --all-namespaces

# 3. Confirm client-side mTLS is actually configured
kubectl get destinationrule --all-namespaces

# 4. Confirm actual live traffic status per workload
istioctl x describe pod <pod-name> -n <namespace>

# 5. Confirm certificate validity across the mesh
istioctl proxy-status
```

`istioctl proxy-status` specifically reports the **sync status** between istiod and every Envoy in the mesh (`SYNCED` vs `STALE`/`NOT SENT`) — a `STALE` entry here means that particular Envoy hasn't received the latest config (including your mTLS policy), which would explain inconsistent behavior on that one pod specifically even though your YAML looks correct.
