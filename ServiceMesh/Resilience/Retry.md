# Istio Retries — The Full Story, From Failed Call to Manifest

*How Service A automatically retries a failed call to Service B, why that's more dangerous than it sounds, and the YAML that controls all of it.*

---

## Table of Contents

1. [The Problem — Why Retries Exist At All](#1)
2. [Where the Retry Actually Happens](#2)
3. [The Request Flow, Call by Call](#3)
4. [Retries vs Circuit Breaking — How the Two Interact](#4)
5. [The Full Annotated Manifest](#5)
6. [The Danger Nobody Warns You About: Retry Storms](#6)
7. [Retries at the App Layer vs the Mesh Layer](#7)
8. [Idempotency — the Question You Must Ask Before Enabling Any Retry](#8)
9. [Putting It All Together — the Mental Model](#9)
10. [Common Mix-Ups, Cleared Up](#10)

---

<a name="1"></a>
## 1. The Problem — Why Retries Exist At All

Picture the same two services from before: Service A calls Service B, which is backed by three pods. Most failures in a distributed system aren't permanent — they're transient blips. A pod briefly garbage-collecting for 200ms, a connection that happened to be stale right when it was reused, a momentary network hiccup between nodes, a pod that's mid-rollout and not quite ready yet. In all of these cases, the exact same request sent a second time, a moment later, would very likely succeed.

Without any retry mechanism, every one of these transient blips becomes a visible failure that propagates straight back to whatever called Service A in the first place — even though, statistically, trying again almost certainly would have worked. Manually writing retry logic into every single service, in every language your organization uses, for every outbound call, is exactly the kind of repetitive infrastructure work your earlier guide's opening section described — and it's error-prone, because retry logic that's slightly wrong (retrying something that shouldn't be retried, or retrying too aggressively) can make outages *worse*, not better, as you'll see in Section 6.

Istio's answer, consistent with everything else in the mesh, is to pull this logic out of your application code and into infrastructure config — specifically, into a `VirtualService`, configured declaratively, enforced by Envoy, with zero code changes in Service A.

---

<a name="2"></a>
## 2. Where the Retry Actually Happens

Just like circuit breaking, retries are enforced entirely on the **calling side** — Service A's own Envoy sidecar — not anywhere near Service B. When Service A's application makes a call to Service B, that call is intercepted by Service A's sidecar before it hits the network, exactly as described throughout your earlier learning.

If that call fails in a way the retry policy considers retryable (covered in Section 5), Service A's sidecar doesn't hand the failure back to the application immediately. Instead, **it quietly attempts the exact same request again**, entirely on its own, without Service A's application code knowing anything happened — unless every attempt eventually fails, at which point the *final* failure is what gets surfaced back to the app.

This is configured through a `VirtualService`, a different object than the `DestinationRule` you used for circuit breaking. The distinction matters: `DestinationRule` configures *policies about a destination* (connection pools, outlier detection, load balancing, subsets). `VirtualService` configures *routing behavior* — where traffic goes, how it's split, and, relevantly here, what happens when a request to a route fails. Retries live in `VirtualService` because a retry is fundamentally a routing decision: "if this attempt failed, route this same request again."

---

<a name="3"></a>
## 3. The Request Flow, Call by Call

Let's walk through this exactly the way you walked through circuit breaking earlier — one request, watched closely, with a retry policy configured for `attempts: 3` and `retryOn: 5xx,connect-failure`.

**The call begins.** Service A's application makes a single logical call — from its code's point of view, it calls `serviceb.get-enrollment()` once, expects one response, and moves on. Everything from here happens inside Service A's Envoy sidecar, invisibly.

**Attempt 1.** Envoy's load balancer picks a pod of Service B — say Pod 2 — and sends the request. Pod 2 happens to be mid-restart and the connection fails outright (`connect-failure`, one of the conditions our policy is configured to retry on). Envoy sees this failure and checks its retry policy: is this failure type retryable, and have we used up our attempt budget? Neither condition stops it, so Envoy immediately tries again — **the application never sees this failure at all.**

**Attempt 2.** Envoy's load balancer picks again — it's free to choose a different pod this time, and often will, since the pool has multiple healthy candidates and Envoy has no obligation to retry against the same pod that just failed. Say it picks Pod 1. This time the request actually reaches Pod 1's application code, gets processed, but Pod 1 is under load and returns a 503. That's covered by our `5xx` condition too, so Envoy counts this as retryable attempt number two, and — since we've configured `attempts: 3` — it still has one attempt left.

**Attempt 3.** Envoy picks again, lands on Pod 3, which is healthy and unburdened, processes the request normally, and returns a clean 200. Envoy immediately forwards that success back to Service A's application. **From the application's perspective, the entire call just looked like one normal request that took slightly longer than usual and eventually succeeded** — it has no idea two earlier attempts silently failed first.

**What if all three attempts had failed?** If Attempt 3 had also failed, Envoy would have exhausted its configured attempt budget, and only *then* would it pass the final failure back to Service A's application code — as a single failed call, the same way it would look without retries configured at all. The application's error handling (or lack of it) kicks in exactly as it would for any other failure.

**Timing, and why `perTryTimeout` matters.** Each individual attempt is also subject to its own timeout — `perTryTimeout` — separate from any overall request timeout you might have configured elsewhere. If Pod 2 in Attempt 1 hadn't failed outright but had instead just hung, not responding at all, Envoy wouldn't wait forever — once `perTryTimeout` elapsed on that attempt, it would treat it as a failure, count it against the retry budget, and move on to the next attempt exactly as if it had received an explicit error. Without this, a single hanging pod could make your "3 retries" take an unbounded amount of total time.

---

<a name="4"></a>
## 4. Retries vs Circuit Breaking — How the Two Interact

These two mechanisms live in different manifest objects (`VirtualService` for retries, `DestinationRule` for circuit breaking), are configured independently, but interact directly at runtime, on the exact same sidecar, over the exact same set of pods.

Walk back through the failure walkthrough from your circuit breaking guide, and now layer retries on top: every failed attempt that lands on a specific pod — say Pod 1 keeps getting picked and keeps failing — **also counts toward that pod's consecutive-5xx tally for outlier detection**, completely independently of the retry logic happening around it. Retries and circuit breaking don't coordinate with each other explicitly, but they share the same underlying signal: real responses from real pods.

This produces a genuinely useful compounding effect. Early on, while Pod 1 is only just starting to degrade, retries mostly paper over the problem — a request that first lands on struggling Pod 1 gets silently retried onto healthy Pod 2 or 3, and the caller never even notices anything was wrong. But as Pod 1's failures keep accumulating across many different requests' retry attempts, its outlier detection counter keeps climbing in the background too — and once it crosses the ejection threshold, Pod 1 gets pulled out of the pool entirely. From that point on, **retries stop even considering Pod 1 as a candidate**, because the load balancer's eligible pool no longer includes it — meaning your retries become more effective (no more attempts wasted on a pod known to be bad) at exactly the moment it would otherwise start actively hurting you.

There's also a more subtle protective relationship worth knowing: `DestinationRule`'s `connectionPool.http.maxRetries` field acts as a **ceiling across all in-flight retries to a destination at once**, at the proxy level — a safety valve independent of the per-request `attempts` count configured in `VirtualService`, which exists specifically to prevent the scenario covered next.

---

<a name="5"></a>
## 5. The Full Annotated Manifest

Here's a complete, realistic `VirtualService` implementing everything described above, with every field explained.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: serviceb-retry-policy
  namespace: default
spec:
  hosts:
    - serviceb.default.svc.cluster.local
  http:
    - route:
        - destination:
            host: serviceb.default.svc.cluster.local
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,connect-failure,refused-stream
        retryRemoteLocalities: false
      timeout: 10s
```

**`hosts`** — same pattern as `DestinationRule`'s `host` field: this policy applies to calls made to Service B, and gets compiled and pushed to every calling sidecar exactly the way circuit breaker config does, via Istiod and xDS.

**`http.route.destination.host`** — the actual routing target for matching requests. A `VirtualService` can define multiple `route` blocks matched by path, headers, or other conditions, each with its own independent retry policy — this example keeps it simple with one route covering everything.

**`retries.attempts`** — the maximum number of *retry* attempts, not counting the original request. With `attempts: 3`, a request can be sent up to four times total (1 original + 3 retries) before Envoy gives up and surfaces the failure to the application, exactly as walked through in Section 3.

**`retries.perTryTimeout`** — how long Envoy waits for a response on any single attempt before treating it as a failed attempt and moving on to the next retry, as covered in Section 3's timing discussion. This should generally be shorter than the overall `timeout` field, since you want room for at least a couple of attempts within your total budget.

**`retries.retryOn`** — the specific conditions that qualify a failure as retryable, as a comma-separated list. Common values:

- `5xx` — any 5xx response code from the upstream.
- `gateway-error` — a narrower subset: specifically 502, 503, and 504.
- `connect-failure` — the connection to the upstream couldn't even be established (as in Attempt 1 of the walkthrough).
- `refused-stream` — the upstream reset the request before processing it, common with HTTP/2.
- `retriable-4xx` — a much narrower, less commonly used condition covering specific 4xx codes that genuinely can indicate a transient issue (like 409 in some contexts) rather than a client error that retrying can't fix.
- `reset` — the connection was reset mid-request.

Note what's conspicuously absent from safe defaults: plain **429 (Too Many Requests) is not typically included**, because immediately retrying a rate-limited request tends to make the rate-limiting problem worse, not better — if you do want to retry on it, it needs deliberate thought about backoff, which brings us to the next section.

**`retries.retryRemoteLocalities`** — controls whether Envoy is allowed to retry against pods in a different failure domain (a different zone or region, in a multi-zone mesh setup) if the local zone's pods are the ones failing. Left at its default (`false`) in most single-zone setups, but genuinely useful to flip on in multi-region deployments where a whole zone might be degraded and you'd rather cross a zone boundary than fail outright.

**`timeout`** — the overall budget for the *entire* logical request, covering the original attempt plus every retry combined. This is the outer boundary that prevents "3 retries at `perTryTimeout: 2s` each" from silently taking 6+ seconds if that's longer than the caller further upstream is willing to wait — once `timeout` is hit, Envoy stops retrying regardless of how many attempts remain in the budget, and returns whatever the current failure state is.

---

<a name="6"></a>
## 6. The Danger Nobody Warns You About: Retry Storms

This is the section that makes retries genuinely different from circuit breaking in terms of risk profile — circuit breaking can only ever reduce load on a struggling pod, but a poorly configured retry policy can actively make an outage catastrophically worse. It's worth understanding exactly how.

Imagine Service B's pods are genuinely overloaded — not broken, just receiving more traffic than they can currently handle, so response times are climbing and some requests are timing out. Now, every one of those timed-out or failed requests, across every caller of Service B, triggers a retry. **Those retries are additional requests, on top of the load that was already too much.** If dozens of callers are all independently retrying failed calls to an already-overloaded Service B, the retry traffic itself becomes a meaningful fraction of total load — sometimes the majority of it — actively preventing Service B from ever catching up and recovering. This is called a **retry storm**, and it's a well-documented, recurring cause of major outages across the industry: a system that would have recovered on its own from a brief overload instead gets pinned in a failing state by the very mechanism meant to paper over transient failures.

This is precisely why `connectionPool.http.maxRetries` in `DestinationRule` exists as a hard ceiling on total in-flight retries to a destination — it's a circuit-breaker-style safety valve specifically for retry traffic, independent of the per-request `attempts` count.

A few practical mitigations worth knowing, even though they go beyond the basic manifest above:

- **Keep `attempts` conservative.** Three is a common, reasonable default; going much higher multiplies the worst-case load a struggling service receives from retries alone.
- **Never retry on conditions that indicate overload rather than transient failure**, like 429, without deliberate backoff — retrying immediately on a rate-limit signal is close to the textbook definition of making things worse.
- **Combine retries with circuit breaking, not instead of it** — once outlier detection ejects a bad pod, retries stop wasting attempts on it, which naturally reduces the retry storm risk over time as the mesh's health tracking catches up.
- **Consider retry budgets at the client library or service level** in addition to the mesh — some organizations cap total retry volume as a percentage of overall traffic, rather than per-request attempt counts alone, precisely to bound the worst case across the whole system rather than one caller at a time.

---

<a name="7"></a>
## 7. Retries at the App Layer vs the Mesh Layer

This mirrors exactly the distinction you already worked through for circuit breaking, and it's worth stating explicitly rather than assuming it transfers automatically.

Envoy's retry mechanism, like its circuit breaker, is entirely business-logic-agnostic — it retries based on HTTP status codes and connection-level failures, with no awareness of what the request actually *means*. It cannot decide "retry this GET but never retry that specific POST because it's not idempotent" based on anything beyond what you configure in `retryOn` and which route the `VirtualService` block applies to — and even then, it's making that decision the same way for every request matching that route, not per individual call based on runtime business context.

An application-level retry library, by contrast, sits inside your code, wrapping a specific call, and can make retry decisions with full knowledge of what that call does — including things Envoy fundamentally can't know, like whether a specific business operation is safe to retry at all, which is exactly the question Section 8 exists to answer.

| | Istio/Envoy retries | App-level retry logic |
|---|---|---|
| **Layer** | Network proxy, outside your code | Inside your process, wrapping a specific call |
| **Granularity** | Per route, configured by status code/connection failure | Per business operation, fully custom conditions |
| **Awareness of business meaning** | None — just sees requests/responses | Full — it's your code |
| **Backoff strategy** | Basic, exponential by default (with jitter), not deeply customizable | Fully customizable — you control every aspect |
| **Effort to implement** | YAML config, zero app code | Requires wrapping every retryable call in your code, in every language, per service |
| **Best for** | Blanket protection against transient network/pod-level blips, with zero code | Precise, business-aware decisions about what's actually safe to retry and how |

---

<a name="8"></a>
## 8. Idempotency — the Question You Must Ask Before Enabling Any Retry

This is the single most important concept underlying whether retries are safe at all, at either layer, and it's easy to skip past if you're focused purely on the mechanics.

**Idempotent** means: performing the same operation multiple times has the same effect as performing it once. A `GET` request that just reads data is naturally idempotent — reading the same thing three times instead of once causes no harm. A `PUT` that fully replaces a resource with a given state is typically idempotent too — setting a value to "5" three times in a row leaves it at "5", same as setting it once.

**A `POST` that creates something, or processes a payment, or increments a counter, is often not idempotent.** If Service A sends a "charge this customer $50" request, and the request genuinely reaches Service B, gets processed, and the customer's card actually gets charged — but the *response* confirming that success gets lost on the way back (a network blip on the return path, not the request path) — Envoy has no way to know the charge already succeeded. All it sees is "this attempt didn't get a clean response," which, depending on your `retryOn` configuration, could look exactly like a failure worth retrying. **The retry would then genuinely charge the customer a second time**, because from the network layer's point of view, it has no idea the first attempt's side effect already happened — it only knows the response didn't come back cleanly.

This is precisely why blanket "retry on 5xx" policies are dangerous for non-idempotent operations without additional safeguards, and why this matters so much more for retries than it ever did for circuit breaking — a circuit breaker only ever *prevents* a call from going out; a retry actively *sends an additional call*, and that additional call has real side effects if the operation wasn't safe to repeat.

The standard mitigation, used widely in production systems, is **idempotency keys**: the caller generates a unique key for a given logical operation (say, a UUID for "this specific charge attempt") and sends it along with the request. Service B's application code — not Envoy, this genuinely has to live in your business logic — checks whether it's already processed a request with that exact key, and if so, returns the same result without repeating the side effect. This is entirely outside what `VirtualService` retries can do for you; it requires actual code in Service B, which is exactly the kind of business-aware decision Section 7's comparison table points at as something only the application layer can handle.

**The practical rule of thumb:** enabling mesh-level retries for routes serving genuinely idempotent operations (most `GET`s, well-designed `PUT`s) is close to free safety. Enabling them for routes serving non-idempotent operations (`POST`s that create or charge or increment) without an idempotency-key mechanism already in place on Service B's side is a real, concrete risk of duplicated side effects — worth restricting via narrower `VirtualService` route matching (applying retries only to specific, known-safe paths) rather than one blanket policy across an entire service.

---

<a name="9"></a>
## 9. Putting It All Together — the Mental Model

Zoom all the way out, and the whole system reduces to six ideas, stacked on top of each other:

1. **Retries live on the caller's sidecar**, same as circuit breaking — Service A's Envoy decides, Service B never knows a retry mechanism exists.
2. **Failed attempts are invisible to the application**, unless every configured attempt is exhausted — only the final outcome ever reaches Service A's app code.
3. **Retries and circuit breaking share the same underlying signal** — every failed attempt, retried or not, also feeds a pod's outlier detection tally, so the two mechanisms reinforce each other over time.
4. **Retries can make things worse, not just better**, if a struggling service is overloaded rather than transiently broken — retry storms are a real, well-documented failure mode.
5. **The mesh can't know what's safe to retry** — it only sees status codes and connection outcomes, never business meaning, so it can't tell an idempotent `GET` from a payment-triggering `POST` beyond whatever route matching you configure.
6. **Idempotency is the actual gatekeeper of safety**, and it's something only your application code, not Envoy, can guarantee — retries and idempotency keys are a matched pair, not independent concerns.

---

<a name="10"></a>
## 10. Common Mix-Ups, Cleared Up

- **Retries are configured in `VirtualService`, not `DestinationRule`** — the opposite object from circuit breaking, because a retry is fundamentally a routing decision, not a destination-level policy.
- **A retried request is not free** — it's a genuine additional call against the same struggling service, which is exactly what makes retry storms possible.
- **`attempts: 3` means up to four total tries** (the original plus three retries), not three total.
- **`perTryTimeout` and `timeout` are different budgets** — one bounds a single attempt, the other bounds the entire logical request across every attempt combined.
- **Retries don't coordinate with circuit breaking explicitly**, but they feed the same failure counters, so a pod that keeps failing retried requests will eventually get ejected and stop being a retry target at all.
- **Envoy cannot add a fallback any more than the circuit breaker can** — a failed-after-all-retries request still just returns an error to your application; any fallback logic still belongs one layer up, in your own code.
- **Retrying a non-idempotent operation without an idempotency-key mechanism on the receiving side is a real correctness risk**, not just a performance consideration — this is the one place in this whole file where "just enable it, it's free protection" genuinely does not apply.
