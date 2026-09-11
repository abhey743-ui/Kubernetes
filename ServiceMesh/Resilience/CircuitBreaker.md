# Istio Circuit Breaking — The Full Story, From Request to Manifest

*How Service A actually protects itself from a struggling Service B, told from the first failed call to the YAML that makes it happen.*

---

## Table of Contents

1. [The Setup — Two Services, One Struggling Pod](#1)
2. [Where the Breaker Actually Lives](#2)
3. [The Request Flow, Call by Call](#3)
4. [How the Rule Gets From Your YAML to the Right Sidecar](#4)
5. [The Full Annotated Manifest](#5)
6. [Extra Fields Worth Knowing (Not Strictly the Breaker, But Configured Alongside It)](#6)
7. [Why There's No Fallback at This Layer](#7)
8. [Putting It All Together — the Mental Model](#8)
9. [Common Mix-Ups, Cleared Up](#9)

---

<a name="1"></a>
## 1. The Setup — Two Services, One Struggling Pod

Picture two services running in a Kubernetes cluster with Istio installed: **Service A** and **Service B**. Service A calls Service B constantly — maybe it's hitting `serviceb/get-enrollment`, `serviceb/get-data`, `serviceb/get-students`, whatever routes B exposes. Service B is backed by three pods, because it's been scaled out for load: Pod 1, Pod 2, Pod 3.

Now imagine Pod 1 starts misbehaving. Maybe it's under memory pressure, maybe a background job on that specific pod is eating CPU, maybe it just got unlucky and picked up a bad connection to a downstream database replica. Whatever the cause, Pod 1 starts returning HTTP 500s while Pod 2 and Pod 3 are still completely healthy.

Without any protection, here's what naturally happens: Service A keeps sending roughly a third of its traffic to Pod 1 (assuming round-robin load balancing across three pods), every one of those requests fails, threads or connections on Service A's side pile up waiting on a slow/failing dependency, and in the worst case this cascades — Service A itself starts slowing down or running out of resources because it's stuck waiting on a pod that was never going to answer properly.

This is exactly the failure mode circuit breaking exists to prevent. The idea, borrowed directly from electrical circuit breakers, is simple: **once something downstream starts failing repeatedly, stop sending it more load, so it has a chance to recover, and so you stop wasting your own resources on requests that are very likely to fail anyway.**

---

<a name="2"></a>
## 2. Where the Breaker Actually Lives

Here's the detail almost everyone misses at first, and it changes how you think about everything that follows: **the circuit breaker does not live on Service B, the struggling side. It lives on Service A, the calling side.**

Every pod in an Istio mesh runs a second container alongside the application — a sidecar proxy built on **Envoy**. All outbound traffic that Service A's application code sends is transparently intercepted by Service A's own Envoy sidecar before it ever reaches the network. It's this sidecar — sitting right next to Service A's app code, not anywhere near Service B — that decides, for every single outgoing request, "should I actually send this, or is the target I'd send it to currently in bad shape?"

This matters because it flips the mental model people usually bring in from application-level breaker libraries (Hystrix, resilience4j), where you might imagine the breaker sitting "between" the two services in some abstract sense. In Istio, it's concretely, physically sitting on the caller's side, as part of the caller's own infrastructure. Service B never knows a breaker exists. Service B just sees fewer requests arriving at Pod 1 during a bad patch — from B's perspective, that's indistinguishable from Service A simply calling it less.

The configuration that defines the breaker's rules — the thresholds, the timers — is written in an object called a `DestinationRule`, and by convention you write it *targeting the service you're protecting against* (Service B), because that's the natural place to declare "here's how sensitive I want any caller to be about B's health." But as you'll see in Section 4, that YAML doesn't end up living anywhere near Service B either — it gets copied out and enforced entirely on the calling side.

---

<a name="3"></a>
## 3. The Request Flow, Call by Call

This is the part that's easiest to understand as a narrative rather than a diagram, so let's walk through it exactly as it happens on the wire. Assume the breaker is configured with `consecutive5xxErrors: 5` — meaning five failures in a row from the same pod trips it.

**Call 1.** Service A's application code makes a completely normal request to `serviceb/get-enrollment`. This request is intercepted by Service A's Envoy sidecar, which currently sees all three pods of Service B as healthy, so its load balancer picks one — say it picks Pod 1. The request goes out over the network, Pod 1 processes it (badly, because it's struggling) and returns a 500. Envoy passes that 500 straight back to Service A's application, which experiences it as a normal failed call — nothing unusual from the app's point of view. But quietly, in the background, Envoy increments an internal counter it maintains specifically for Pod 1: one consecutive failure recorded.

**Calls 2 through 4.** The same pattern repeats. Some of these might land on Pod 1 again (still in the healthy pool, still eligible to be picked), some might land on Pod 2 or 3 depending on the load balancing algorithm. Every time a request happens to land on Pod 1 and fails, its counter climbs — two, three, four consecutive failures. Requests that land on healthy Pod 2 or Pod 3 succeed normally and have no effect on Pod 1's counter at all; each pod's failure count is tracked completely independently.

**Call 5 — the trip point.** A request lands on Pod 1 one more time, and it fails again — its fifth consecutive failure. The instant that response arrives back at Service A's sidecar, the counter crosses the configured threshold. Right then, without any further requests needed, Envoy removes Pod 1 from its list of pods eligible to receive traffic, and starts a timer — the `baseEjectionTime` — during which Pod 1 will not be sent anything at all. From the application's point of view, this fifth call just looked like one more failed request; nothing about the experience of that specific call was different. The consequence is entirely about what happens to *future* calls.

**Call 6 onward.** Service A's application makes another request. Its sidecar's load balancer now only has Pod 2 and Pod 3 in its eligible pool — Pod 1 isn't even considered as a candidate. The request goes to one of the two healthy pods and succeeds normally. This repeats for every subsequent call during the ejection window: **no traffic reaches Pod 1 at all**, regardless of which specific route on Service B is being called (`/get-enrollment`, `/get-data`, `/get-students` — doesn't matter, the ejection is per-pod, not per-route, because Envoy's breaker has no concept of URL paths, only of which upstream host it's talking to).

**During the ejection window.** If Pod 2 and Pod 3 end up absorbing all the traffic that would have gone to Pod 1, and if connection pool limits (covered in Section 5) are tight, it's possible for Pod 2 or Pod 3 to also start failing under the extra load — and if so, they'd accumulate their own separate consecutive-failure counters, completely independently of Pod 1's. There's no shared or combined health state across pods; each one is tracked on its own.

**When the timer expires.** After `baseEjectionTime` elapses (commonly something like 30 seconds), Pod 1 is allowed back into the candidate pool. Envoy doesn't blindly trust it — the next request or two routed to Pod 1 effectively acts as a probe. If Pod 1 responds successfully, its counter resets to zero and it's treated as fully healthy again, back in normal rotation. If it fails again immediately, it gets re-ejected — and by default, Istio increases the ejection time on repeated offenses (an exponential backoff, up to a configurable maximum), so a persistently broken pod gets probed less and less often rather than being hammered every 30 seconds forever.

**The one thing to hold onto from this whole walkthrough:** there is no message, no heartbeat, no side-channel where Service B tells Service A "I'm unhealthy." Service A's sidecar figures this out purely by watching the ordinary responses to ordinary requests it was already sending as part of its normal job. The breaker is an emergent property of careful bookkeeping on the caller's side, not a conversation between the two services.

---

<a name="4"></a>
## 4. How the Rule Gets From Your YAML to the Right Sidecar

A natural question at this point: if the breaker lives on Service A's sidecar, but you write the `DestinationRule` targeting Service B, how does the rule actually end up where it's enforced?

This is the job of Istio's control plane component, **Istiod**. Here's the sequence:

1. You write a `DestinationRule` YAML file with `host: serviceb` and apply it with `kubectl apply`.
2. This gets stored as a Kubernetes custom resource, watched continuously by Istiod.
3. Istiod compiles this rule into Envoy's native configuration format and pushes it out to every Envoy sidecar in the mesh that has a reason to talk to Service B — using a protocol called **xDS** (specifically the Cluster Discovery Service, or CDS, which is the part of xDS responsible for pushing down cluster-level policy like connection pools and outlier detection).
4. Every sidecar that receives this — Service A's, and any other service's sidecar that also calls Service B — stores it **locally, in its own memory**, as part of its definition of "the Service B cluster."
5. From that point on, when Service A's sidecar decides whether to route to Pod 1, it's comparing its own locally-tracked failure counter against a threshold it already has cached — no live lookup, no round-trip to Istiod or to Service B, just a local comparison against config it received ahead of time.

**This also answers a related question worth stating explicitly: the rule is universal by default.** Because Istiod pushes the identical compiled configuration to every caller of Service B, if a third service (call it Service C) also calls Service B, Service C's sidecar independently enforces the exact same threshold, with its own completely separate failure counters and its own separate view of which of Service B's pods are currently ejected. Service A ejecting Pod 1 has zero effect on what Service C's sidecar decides — they don't share state or coordinate with each other at all; they simply both apply the same rule independently, based only on what each of them personally observes.

If you ever wanted different callers to treat the same destination differently, a single `DestinationRule` doesn't give you that out of the box — you'd need more advanced scoping (Istio's `Sidecar` resource, or per-namespace `DestinationRule` overrides), which is a less common, more advanced setup most teams don't need.

---

<a name="5"></a>
## 5. The Full Annotated Manifest

Here's a complete, realistic `DestinationRule` implementing everything described above, with every field explained.

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: serviceb-circuit-breaker
  namespace: default
spec:
  host: serviceb.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 5s
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 10
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 5
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
      splitExternalLocalOriginErrors: false
```

**`host`** — the service this rule applies to, using its full Kubernetes DNS name. This is the target Envoy will apply the policy for, but as covered in Section 4, the policy actually gets enforced on every *caller's* sidecar, not on Service B's own sidecar.

**`connectionPool.tcp.maxConnections`** — the maximum number of simultaneous TCP connections a single caller's sidecar will open to the entire Service B cluster (across all its pods combined). Once this cap is hit, new connection attempts are queued or rejected rather than piling up indefinitely. This protects the caller from over-committing resources to one dependency, and protects Service B from being overwhelmed by one aggressive caller.

**`connectionPool.tcp.connectTimeout`** — how long Envoy will wait for a TCP connection to actually establish before giving up and treating it as a failure. A short timeout here means a genuinely unreachable pod is detected quickly rather than hanging.

**`connectionPool.http.http1MaxPendingRequests`** — for HTTP/1.1 traffic, the maximum number of requests allowed to sit queued waiting for a connection to become available, before Envoy starts rejecting new requests outright with a fast failure instead of making them wait.

**`connectionPool.http.http2MaxRequests`** — the equivalent cap for HTTP/2 traffic, which multiplexes many requests over fewer connections, so this is expressed as a request count rather than a connection count.

**`connectionPool.http.maxRequestsPerConnection`** — after this many requests have gone over a single connection, Envoy closes it and opens a fresh one. This exists partly for load-balancing hygiene (long-lived connections can get "stuck" favoring one backend pod disproportionately) and partly to bound how much state accumulates on a single connection.

**`connectionPool.http.maxRetries`** — the maximum number of retries Envoy itself will attempt across all in-flight requests to this destination, at the proxy level. This is separate from — and does not replace — any retry policy you might also configure in a `VirtualService`.

**`outlierDetection.consecutive5xxErrors`** — the exact threshold you configured throughout the walkthrough in Section 3: how many consecutive 5xx responses from a single pod trip the breaker for that pod specifically.

**`outlierDetection.consecutiveGatewayErrors`** — a related but distinct counter, specifically for gateway-style errors (502, 503, 504), which often indicate connectivity or upstream availability problems rather than application-level failures. You can tune this separately from `consecutive5xxErrors` if you want different sensitivity for "the app returned an error" versus "the network/proxy layer couldn't even reach it properly."

**`outlierDetection.interval`** — how frequently Envoy sweeps its pool and evaluates ejection/recovery decisions. This is the "tick rate" of the whole mechanism, not a per-request check — Envoy isn't recalculating on every single response, it's periodically evaluating state at this interval.

**`outlierDetection.baseEjectionTime`** — the walkthrough's ejection window: how long a pod stays out of the pool once ejected, before being given a chance to rejoin. As mentioned in Section 3, this increases automatically on repeated ejections of the same pod (up to a configurable maximum), so it's a *base*, not a fixed constant across all ejections.

**`outlierDetection.maxEjectionPercent`** — a safety ceiling: no matter how many pods start failing, Envoy will never eject more than this percentage of the total pool at once. This exists specifically to prevent a scenario where, say, all three pods of Service B briefly wobble at the same time, and the breaker accidentally ejects every single one, leaving zero pods available and turning a temporary blip into a total outage. With `maxEjectionPercent: 50` and three pods, at most one (or in some rounding cases, one and a half rounded down) gets ejected at a time even if all three are technically failing the threshold.

**`outlierDetection.minHealthPercent`** — a related safety mechanism: if the percentage of currently-healthy pods drops below this value, Envoy stops applying outlier detection ejections altogether and instead sends traffic to all pods, healthy or not. The reasoning is blunt but sound: if most of your pods are already unhealthy, ejecting more of them doesn't help — you're better off spreading load across everything you have rather than concentrating it onto an even smaller remaining pool.

**`outlierDetection.splitExternalLocalOriginErrors`** — a more advanced flag controlling whether Envoy distinguishes between errors that originated locally (in Envoy itself — like a connection timeout) versus errors that came back from the actual upstream service (like an application-returned 500). Left at its default (`false`), both types are counted together toward the same consecutive-error tally.

---

<a name="6"></a>
## 6. Extra Fields Worth Knowing (Not Strictly the Breaker, But Configured Alongside It)

These aren't part of circuit breaking itself, but they live in the same `trafficPolicy` block and are genuinely useful to know about when you're already in this file writing breaker config, since they shape how the breaker's decisions actually play out in practice.

**`loadBalancer.simple`** — controls the algorithm used to pick among the currently-healthy pods: `ROUND_ROBIN`, `LEAST_CONN`, `RANDOM`, or `PASSTHROUGH`. This matters for the breaker because it determines how traffic gets redistributed onto the surviving pods once one gets ejected — `LEAST_CONN`, for instance, actively favors sending traffic to whichever healthy pod currently has the fewest active requests, which can help avoid accidentally overloading Pod 2 and Pod 3 right after Pod 1 gets ejected.

```yaml
trafficPolicy:
  loadBalancer:
    simple: LEAST_CONN
```

**`subsets`** — lets you define named groups of pods (typically by version label, like `v1` and `v2`) and apply *different* traffic policies, including different breaker thresholds, to each subset independently. This is genuinely useful if, say, a canary version of Service B is expected to be less stable during rollout and you want a more aggressive (lower threshold, shorter ejection time) breaker just for that subset, while the stable version keeps a more relaxed one.

```yaml
subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      outlierDetection:
        consecutive5xxErrors: 2
```

**`tls.mode`** — while unrelated to breaking directly, this is commonly configured in the same `DestinationRule` since you're already touching `trafficPolicy` for this destination. Setting `ISTIO_MUTUAL` here is what actually turns on mTLS for calls to this destination, using the certificates the mesh's control plane automatically issues and rotates.

None of these are required for a basic circuit breaker to work — the manifest in Section 5 is complete on its own — but they're the fields you'll realistically reach for next once the basic breaker is working and you want to fine-tune how traffic behaves around it.

---

<a name="7"></a>
## 7. Why There's No Fallback at This Layer

It's worth explicitly naming something that trips people up coming from application-level circuit breaker libraries like Hystrix or resilience4j: **there is no fallback function anywhere in this manifest, and that's not an oversight — it's a structural limitation of where this breaker lives.**

An application-level breaker sits inside your process, wrapping a specific function call, and when it trips, it invokes *your own code* — which can do anything your business logic needs: return cached data, queue the request for later, degrade gracefully, whatever makes sense for that specific operation.

Envoy's breaker has no equivalent option, because it operates purely as a network proxy with no knowledge of what your requests *mean*. It doesn't know that `/get-enrollment` is semantically different from `/get-data`, and it has no concept of "acceptable degraded response" versus "actual failure" beyond HTTP status codes. When it trips, the only thing it's capable of doing is what a proxy can do: stop forwarding the request and return an error (typically a 503) straight back to the caller's application code, instantly, without ever hitting the network.

That returned 503 lands in Service A's application exactly like any other failed call would — the app genuinely can't distinguish "Envoy short-circuited this before it ever left the building" from "Service B itself returned a 503." This is deliberate: the app doesn't need to know *why* a call failed, only that it did, and any fallback behavior you want beyond "stop hammering a bad pod" belongs one layer up, in your own application code wrapping that outbound call.

---

<a name="8"></a>
## 8. Putting It All Together — the Mental Model

Zoom all the way out, and the whole system reduces to five ideas, stacked on top of each other:

1. **The breaker lives on the caller, not the callee.** Service A's own sidecar makes every decision; Service B never knows a breaker exists.
2. **Health is tracked per pod, not per route.** A failing `/get-enrollment` call on Pod 1 counts against Pod 1 as a whole — every route to that pod gets blocked once it's ejected, not just the one that was failing.
3. **There's no live communication for the rule itself.** The threshold (5 consecutive errors, 30-second ejection, whatever you configure) is pushed once, ahead of time, by Istiod to every caller's sidecar, and lives there as local config — checked against a locally-tracked counter, with zero network round-trips per decision.
4. **The rule is universal across all callers by default.** Every service that calls Service B gets the identical policy, enforced with completely independent state — no shared visibility between different callers' breakers.
5. **The only action available when it trips is "stop and return an error."** No fallback, no business logic, no graceful degradation — that's the job of your application code, one layer above the mesh.

---

<a name="9"></a>
## 9. Common Mix-Ups, Cleared Up

- **"Endpoint" in the breaker context means a pod, not a URL path.** This is the single most common confusion, and almost every other misunderstanding traces back to it.
- **The breaker isn't a message exchanged between services.** It's one side quietly counting outcomes it was already observing during normal traffic.
- **A `DestinationRule` targeting Service B doesn't run on Service B.** It gets compiled and distributed to every *caller's* sidecar and enforced there.
- **Ejecting a pod isn't permanent, and it isn't instant to reverse either.** There's a timer, a probe-based recovery, and exponential backoff on repeat offenders — it's a temporary, self-healing state, not a one-way switch.
- **Different services calling the same destination don't share breaker state.** Each caller's sidecar tracks its own view of the world independently, even though they're all enforcing the same rule.
- **There is genuinely no fallback mechanism available at this layer**, and reaching for one here means you're looking for the wrong tool — that belongs in your application code, wrapping the specific call, where business context actually exists.
