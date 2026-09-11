# Enabling the Mesh — Installing Istio, Step by Step

*How a plain Kubernetes cluster with zero mesh awareness turns into one where Service A and Service B are talking through sidecars, told from first command to first verified request.*

---

## Table of Contents

1. [What "Enabling the Mesh" Actually Means](#1)
2. [Prerequisites — What Needs to Exist First](#2)
3. [Installing istioctl and Istio Itself](#3)
4. [Choosing an Installation Profile](#4)
5. [The Actual Install, Command by Command](#5)
6. [Turning On Sidecar Injection — the Step That Actually Matters](#6)
7. [Deploying a Real App and Watching the Sidecar Appear](#7)
8. [Verifying the Mesh Is Actually Doing Something](#8)
9. [Getting Traffic Into the Mesh — the Ingress Gateway](#9)
10. [The Observability Add-Ons](#10)
11. [Turning Existing Services Into Mesh Members Safely](#11)
12. [Common Problems and How to Read Them](#12)
13. [Putting It All Together — the Mental Model](#13)

---

<a name="1"></a>
## 1. What "Enabling the Mesh" Actually Means

Everything you've learned so far — circuit breaking, retries, per-pod outlier detection, all of it — depends on one precondition: **every pod involved needs an Envoy sidecar sitting next to it, and a control plane (Istiod) needs to be running somewhere in the cluster to configure those sidecars.** "Enabling the mesh" is really just the process of getting those two things true.

Concretely, that breaks down into two separate actions, and it's worth keeping them mentally separate because they happen at different times and different scopes:

1. **Installing Istio's control plane** — a one-time, cluster-wide action. You run this once (or once per cluster), and it deploys Istiod along with whatever supporting components your chosen profile includes.
2. **Enabling sidecar injection for specific namespaces** — a per-namespace, ongoing decision. This is the switch that actually determines *which* of your workloads get a sidecar and join the mesh. A cluster can have Istio installed cluster-wide while only a handful of namespaces actually participate.

This distinction matters because it means installing Istio doesn't instantly mesh-ify your entire cluster — nothing changes for any existing workload until you explicitly opt a namespace in, which is deliberate and gives you control over the rollout.

---

<a name="2"></a>
## 2. Prerequisites — What Needs to Exist First

Before touching Istio at all, you need:

- **A running Kubernetes cluster** you have `kubectl` access to, with cluster-admin permissions (installing Istio creates CRDs — Custom Resource Definitions — and cluster-scoped resources, which requires elevated privileges).
- **A supported Kubernetes version.** Istio tracks Kubernetes releases closely and generally supports the current and a few recent minor versions — worth checking Istio's own compatibility matrix for your specific cluster version rather than assuming.
- **Enough cluster resources.** Istiod itself, plus a sidecar per pod, adds real CPU/memory overhead across the cluster — this was flagged as a genuine trade-off back in your original guide's Section 5, and it's worth confirming your cluster has headroom before installing, especially on a small local cluster like kind or minikube.
- **A local machine to run `istioctl` from** — this is the CLI tool used to install and manage Istio, and it talks to your cluster the same way `kubectl` does, using your existing kubeconfig.

If you're learning on a local cluster, `kind` or `minikube` both work fine for everything in this guide — just be aware resource limits bite harder locally than they would on a proper cloud cluster.

---

<a name="3"></a>
## 3. Installing istioctl and Istio Itself

`istioctl` is a standalone binary, separate from `kubectl`, and it's the primary tool for installing and inspecting Istio. The simplest way to get it:

```bash
curl -L https://istio.io/downloadIstio | sh -
```

This downloads the latest release into a versioned directory in your current folder (something like `istio-1.24.0/`). Add its `bin` directory to your `PATH` so `istioctl` is available globally:

```bash
export PATH="$PWD/istio-1.24.0/bin:$PATH"
```

Verify it's working and check which version you have:

```bash
istioctl version
```

At this point, nothing has touched your cluster yet — you've only installed the local tool that will do the installing.

---

<a name="4"></a>
## 4. Choosing an Installation Profile

Istio ships with several pre-built **profiles** — bundles of components and default settings suited to different purposes. Picking the right one matters, because it determines what actually gets deployed into your cluster.

- **`default`** — the profile most production clusters start from. Installs Istiod and a default ingress gateway, with production-sensible resource requests. This is the one you'd typically build on and customize further.
- **`demo`** — installs everything, including both ingress and egress gateways, with all tracing/telemetry sampling turned way up and resource requests kept intentionally low. This is the profile almost every official tutorial and this file's own examples assume, because it's built specifically for learning and experimentation, not production efficiency.
- **`minimal`** — installs just the control plane (Istiod) and nothing else, useful if you want to add gateways and other pieces separately and explicitly.
- **`ambient`** — installs the newer sidecar-less architecture (ztunnel + optional waypoint proxies) instead of the classic sidecar model. Worth knowing exists, but per your earlier guide's own recommended learning path, sidecar mode's concepts are worth learning first since they transfer directly.

For everything that follows in this guide, we'll use the **`demo`** profile, since it's the standard choice for learning and matches what you'll see in the official Bookinfo tutorial and most walkthroughs.

---

<a name="5"></a>
## 5. The Actual Install, Command by Command

**Step 1 — Install the control plane.**

```bash
istioctl install --set profile=demo -y
```

The `-y` flag skips the interactive confirmation prompt. Running this without it first is worth doing once, though, since `istioctl` will show you exactly what it's about to install before you confirm — a genuinely useful way to see the full component list for your chosen profile before committing.

**What this actually does under the hood:** it creates a new namespace called `istio-system`, deploys Istiod into it (along with, for the `demo` profile, ingress and egress gateway deployments), and installs a set of CRDs into your cluster — these are what make Kubernetes understand `DestinationRule`, `VirtualService`, `Gateway`, `PeerAuthentication`, and every other Istio-specific object type you've configured in earlier guides. Without these CRDs installed, `kubectl apply` on any of those YAML files would simply fail with "no matches for kind."

**Step 2 — Verify the control plane came up healthy.**

```bash
kubectl get pods -n istio-system
```

You should see `istiod` in a `Running` state, along with `istio-ingressgateway` and `istio-egressgateway` pods (for the `demo` profile specifically — a `minimal` install would show just `istiod`).

**Step 3 — Run Istio's own built-in health check.**

```bash
istioctl verify-install
```

This checks that everything the profile was supposed to install is actually present and correctly configured in the cluster, and will flag anything that's missing or misconfigured before you move on.

At this point, Istio's control plane is running — but **nothing has changed for any of your existing workloads yet.** No sidecars have been injected anywhere. This is the deliberate boundary from Section 1: installation and sidecar injection are separate steps.

---

<a name="6"></a>
## 6. Turning On Sidecar Injection — the Step That Actually Matters

This is the single command that transforms a namespace from "ordinary Kubernetes" into "part of the mesh":

```bash
kubectl label namespace default istio-injection=enabled
```

That's it — one label, on one namespace. From this point forward, **any new pod created in that namespace** will automatically get an Envoy sidecar container injected into it, alongside your application container.

**Why "new pod," specifically — and not existing ones.** Sidecar injection happens through a Kubernetes feature called a **mutating admission webhook**. When a pod is *created*, Kubernetes calls out to Istiod (which registers itself as this webhook) and asks, "does this pod need anything added before it's finalized?" Istiod checks whether the pod's namespace carries the `istio-injection=enabled` label, and if so, it modifies the pod spec on the fly — adding the Envoy container, and the `initContainer` that sets up the iptables redirect rules — before the pod actually starts running.

This mechanism only fires at pod **creation** time. Any pod that was already running before you applied the label is completely unaffected — it keeps running exactly as it was, with no sidecar, because the webhook never got a chance to intercept its creation. This is precisely why the next section matters.

**Verify the label took:**

```bash
kubectl get namespace default --show-labels
```

You should see `istio-injection=enabled` in the label list.

---

<a name="7"></a>
## 7. Deploying a Real App and Watching the Sidecar Appear

To actually see sidecar injection happen, deploy something new into your now-labeled namespace. The standard example used across almost every official Istio tutorial is **Bookinfo**, a small multi-service sample app specifically built to demonstrate mesh features — it's the same app referenced in your earlier guide's learning path.

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml
```

Now check what actually got created:

```bash
kubectl get pods
```

The key thing to look for: each pod should show **`2/2`** containers ready, not `1/1`. That second container is the injected Envoy sidecar. You can confirm this directly:

```bash
kubectl describe pod <pod-name>
```

Under `Containers:`, you'll see your application container's name as expected, plus a second container named `istio-proxy` — that's Envoy. There's also usually an `Init Containers:` section showing `istio-init`, the short-lived container that runs once at pod startup specifically to set up the iptables rules that transparently redirect traffic into the sidecar, exactly as described throughout your earlier guides.

**If you deployed Bookinfo before labeling the namespace** (or into a namespace you forgot to label), you'll see `1/1` instead — no sidecar, because the webhook never fired for that pod's creation. The fix isn't to somehow inject a sidecar into a running pod; you simply label the namespace correctly (if not already done) and then force pod recreation, since injection only happens at creation time:

```bash
kubectl rollout restart deployment <deployment-name>
```

This triggers Kubernetes to create fresh pods for that deployment, which now get intercepted by the webhook and receive sidecars, replacing the old sidecar-less ones.

---

<a name="8"></a>
## 8. Verifying the Mesh Is Actually Doing Something

Having sidecars present is necessary but not sufficient proof the mesh is functioning correctly — worth confirming a few more things.

**Check Istiod is actually pushing config to your sidecars:**

```bash
istioctl proxy-status
```

This lists every sidecar Istiod currently knows about, along with a sync status column. You want to see `SYNCED` for every proxy — anything showing `STALE` or `NOT SENT` indicates Istiod isn't successfully communicating with that particular sidecar, worth investigating before trusting any traffic policy you apply.

**Inspect what a specific sidecar actually has configured**, which is genuinely useful once you start applying your own `DestinationRule`/`VirtualService` objects from earlier guides and want to confirm they actually landed:

```bash
istioctl proxy-config cluster <pod-name>
```

This dumps Envoy's live cluster configuration for that specific pod — you can search this output for your destination service's name and confirm settings like `maxConnections` or outlier detection thresholds are actually present, rather than just trusting that `kubectl apply` succeeded.

**Run Istio's built-in config sanity checker**, which catches a wide range of common misconfigurations (a `VirtualService` referencing a host with no matching `DestinationRule`, conflicting routes, and similar issues) before they cause confusing runtime behavior:

```bash
istioctl analyze
```

---

<a name="9"></a>
## 9. Getting Traffic Into the Mesh — the Ingress Gateway

Everything so far covers service-to-service traffic *inside* the mesh — but real traffic has to enter from somewhere outside the cluster first. This is the job of the **ingress gateway**, deployed automatically as part of the `demo` profile install in Section 5.

Unlike the sidecars, which are injected per-pod, the ingress gateway is itself a standalone Envoy deployment sitting at the mesh's edge, and you point traffic at it using a `Gateway` object (defining what ports/hosts it accepts) paired with a `VirtualService` (defining how that incoming traffic gets routed to services inside the mesh):

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - "*"
  gateways:
    - bookinfo-gateway
  http:
    - match:
        - uri:
            prefix: /productpage
      route:
        - destination:
            host: productpage
            port:
              number: 9080
```

**`Gateway.selector`** — picks which gateway deployment this configuration applies to, matched by label (`istio: ingressgateway` targets the default ingress gateway that came with your profile).

**`Gateway.servers`** — defines which ports and hosts the gateway will accept traffic for. `hosts: ["*"]` here accepts any hostname, fine for local testing, but worth narrowing to your actual domain in anything resembling production.

**`VirtualService.gateways`** — this is the field that ties a `VirtualService` to a `Gateway` rather than to mesh-internal traffic. Without this field, a `VirtualService` only applies to traffic already inside the mesh (as in your retry and circuit breaker examples); including it here means this routing rule also governs traffic arriving from outside, through that specific gateway.

Find the gateway's actual external address and test it:

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

On a cloud cluster this typically shows an external `LoadBalancer` IP; on a local `kind`/`minikube` cluster you'll usually need port-forwarding instead:

```bash
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

Then a request to `http://localhost:8080/productpage` should reach Bookinfo's front-end service, having passed through the ingress gateway's Envoy first.

---

<a name="10"></a>
## 10. The Observability Add-Ons

The `demo` profile installs the control plane and gateways, but the dashboards referenced throughout your earlier guides — Kiali, Grafana, Prometheus, Jaeger — are separate, optional add-ons, installed from manifests bundled in the same Istio release download:

```bash
kubectl apply -f samples/addons
```

(Run from inside the downloaded `istio-1.24.0/` directory from Section 3, where the `samples/addons` folder lives.)

Wait for these to come up, then check them:

```bash
kubectl get pods -n istio-system
```

You should now see `kiali`, `grafana`, `prometheus`, and `jaeger` (or a similar tracing backend) pods alongside `istiod` and the gateways. Open Kiali specifically — the tool that visualizes your mesh's live topology, referenced back in your original service mesh guide's Section 4.3 — with:

```bash
istioctl dashboard kiali
```

This opens a browser tab automatically, port-forwarded to the Kiali service, and once you've generated some traffic through Bookinfo (refresh `/productpage` a few times), you'll see the actual service graph render live — which is genuinely one of the more satisfying moments in learning the mesh, since it's the first time all the invisible sidecar-to-sidecar traffic becomes something you can literally look at.

---

<a name="11"></a>
## 11. Turning Existing Services Into Mesh Members Safely

Section 6 covered turning on injection for a namespace with nothing running in it yet — a clean starting point. Real clusters usually already have workloads running, and rolling the mesh out onto them safely deserves a slightly more careful approach than "label the namespace and restart everything at once."

**A more gradual, safer rollout pattern:**

1. Label the target namespace as in Section 6.
2. Restart deployments **one at a time**, watching for `2/2` and healthy readiness after each, rather than restarting every deployment in the namespace simultaneously.
3. Check `istioctl proxy-status` after each restart to confirm the new sidecar synced correctly before moving to the next service.
4. Leave `PeerAuthentication` in `PERMISSIVE` mode initially (mentioned in your TLS/mTLS guide's Section 11) rather than `STRICT`, so services that haven't been migrated into the mesh yet can still talk to ones that have, without mTLS being forcibly required on connections where one side doesn't have a sidecar yet.
5. Only tighten to `STRICT` mTLS once every service that needs to talk to the ones you've migrated is itself inside the mesh.

This staged approach avoids the failure mode of restarting an entire namespace's workloads at once and discovering, only after the fact, that some untouched dependency outside the mesh can no longer reach a service that now expects mTLS.

---

<a name="12"></a>
## 12. Common Problems and How to Read Them

**Pods stuck at `1/1` after labeling the namespace.** Almost always means the pods were created *before* the label was applied — see Section 7's fix (restart the deployment). Double-check the label is actually on the namespace the pod lives in, not a similarly-named one.

**`istioctl proxy-status` shows `STALE` for a proxy.** Usually indicates a connectivity problem between that sidecar and Istiod — check that the pod can actually reach `istiod.istio-system.svc` on the expected port, and that no overly restrictive `NetworkPolicy` is blocking that traffic.

**`kubectl apply` on a `DestinationRule`/`VirtualService`/etc. fails with "no matches for kind."** Means the Istio CRDs aren't installed — confirm the install in Section 5 actually completed successfully with `istioctl verify-install` before assuming your own YAML is at fault.

**A service works fine without the mesh but breaks immediately after sidecar injection.** Very commonly a `PeerAuthentication` mismatch — one side expects mTLS, the other doesn't have a sidecar yet and can't speak it. Section 11's staged `PERMISSIVE`-first rollout exists specifically to avoid this.

**Traffic reaches the ingress gateway but never makes it to the target service.** Check that the `VirtualService`'s `gateways` field actually references your `Gateway`'s name correctly, and that the `host`/port referenced in the destination match the target service's actual Kubernetes service name and port exactly — a common typo source, especially with the full DNS form (`servicename.namespace.svc.cluster.local`) versus the short form.

---

<a name="13"></a>
## 13. Putting It All Together — the Mental Model

Six ideas, in the order you'd actually do them:

1. **Installing Istio and enabling injection are two separate actions** — installing gives you a control plane; nothing about your workloads changes until you explicitly label a namespace.
2. **Sidecar injection only ever happens at pod creation**, via a mutating admission webhook — labeling a namespace never retroactively touches already-running pods; you have to force recreation.
3. **A `2/2` (or higher) container count on a pod is your confirmation a sidecar is present** — `describe pod` shows you the injected `istio-proxy` container and the `istio-init` init container directly.
4. **`istioctl proxy-status` and `istioctl analyze` are your first stops whenever something's not working** — they check sync health and config correctness before you go digging into individual sidecar configs.
5. **Traffic entering from outside the mesh needs a `Gateway` plus a `VirtualService` with `gateways` set** — mesh-internal `VirtualService` objects (like your retry policy) never touch external traffic on their own.
6. **Rolling the mesh onto existing production workloads deserves a staged approach** — one service at a time, `PERMISSIVE` mTLS first, tightened only once everything that needs to talk to a migrated service is itself inside the mesh.
