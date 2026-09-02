# 4. ServiceAccount

## The exact YAML

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

## Line by line

| Line | Meaning |
|---|---|
| `apiVersion: v1` | ServiceAccounts are part of the core Kubernetes API. |
| `kind: ServiceAccount` | Tells Kubernetes: create an **identity object**, not a config or workload object. |
| `metadata.name: admin-user` | The name of this identity. Anything that wants to "be" this ServiceAccount, or grant it permissions, refers to it by this name. |
| `metadata.namespace: kubernetes-dashboard` | The **namespace** this ServiceAccount lives in — explained below. |

That's the entire object. It's deliberately tiny — a ServiceAccount by itself does almost nothing. It becomes meaningful only once combined with a token (note 5) and permissions (note 6).

## What is a ServiceAccount?

A **ServiceAccount** is an **identity inside Kubernetes** — but specifically an identity meant for **non-human actors**: Pods, applications, scripts, dashboards, controllers, and other software that needs to talk to the Kubernetes API Server.

Compare it to human identity:

```
Human user   -> logs in with a username/password or SSO -> has an identity
Pod/App      -> uses a ServiceAccount + token             -> has an identity
```

Every Pod in Kubernetes actually runs *as* some ServiceAccount, whether you specify one or not (if you don't specify one, it uses the `default` ServiceAccount of its namespace).

## ServiceAccount as a Kubernetes identity

Think of a ServiceAccount purely as **"a name Kubernetes recognizes."** By itself, this name:

- Has **no permissions** attached automatically.
- Has **no way to prove who it is** until it also has a token.

```
ServiceAccount "admin-user"
        |
        |  (on its own, right now)
        v
   Just a name. No powers. No proof of identity yet.
```

Permissions and proof-of-identity are added by two **separate** pieces, which is exactly why this topic feels confusing at first:

- **Proof of identity** comes from a **Token** (note 5).
- **Permissions** come from **RBAC** — specifically a ClusterRoleBinding here (note 6).

## What is a namespace?

A **namespace** is a way of **dividing a Kubernetes cluster into separate logical sections**, so different teams, applications, or environments don't collide or interfere with each other.

```
Kubernetes Cluster
  |
  |-- namespace: kubernetes-dashboard   (dashboard's own objects live here)
  |-- namespace: eazybank                (your microservices could live here)
  |-- namespace: kube-system             (Kubernetes' own internal components)
```

Key points about namespaces:

- Object **names only need to be unique within a namespace**, not across the whole cluster. So you could have a ServiceAccount named `admin-user` in `kubernetes-dashboard` AND a completely different ServiceAccount named `admin-user` in another namespace — Kubernetes treats them as totally separate objects.
- Many permission rules (like a plain `RoleBinding`) are scoped to a single namespace. A `ClusterRoleBinding` (used here) is the exception — it applies **cluster-wide**, across all namespaces (more on this in note 6).
- The `kubernetes-dashboard` namespace here tells us this ServiceAccount was specifically created to be used **by the Kubernetes Dashboard application**, to control what the person logged into the dashboard can see and do.

## Quick recap

- A ServiceAccount = an identity for software (Pods, apps, dashboards), not for humans.
- By itself, it has no permissions and no way to prove itself — those come separately.
- A namespace is a labeled section of the cluster that keeps this identity organized and scoped.
