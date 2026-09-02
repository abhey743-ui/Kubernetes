# 7. Naming Confusion: Why Everything Is Called "admin-user"

If you look back across the last three notes, you'll notice the name `admin-user` shows up **three separate times**, on three **completely different kinds of objects**:

```yaml
# 1) ServiceAccount
metadata:
  name: admin-user            <-- ServiceAccount's own name

# 2) Secret (token)
metadata:
  name: admin-user            <-- Secret's own name
annotations:
  kubernetes.io/service-account.name: "admin-user"   <-- points to the ServiceAccount

# 3) ClusterRoleBinding
metadata:
  name: admin-user            <-- ClusterRoleBinding's own name
subjects:
  - name: admin-user           <-- points to the ServiceAccount
roleRef:
  name: cluster-admin          <-- points to the ClusterRole (different name!)
```

This is a **very common source of confusion** for people learning Kubernetes RBAC, because it looks like `admin-user` is one single thing wired together everywhere. It isn't — it's three unrelated objects that all happen to have been **given the same label** for human convenience.

## Kubernetes does NOT require these to have the same name

There is **no rule** that says a ServiceAccount, its token Secret, and its ClusterRoleBinding must share a name. You could rewrite the exact same setup with three completely different names, and it would work **identically**:

```yaml
# ServiceAccount
metadata:
  name: dashboard-identity

---
# Secret
metadata:
  name: dashboard-login-token
annotations:
  kubernetes.io/service-account.name: "dashboard-identity"   # <-- must match the SA's real name
type: kubernetes.io/service-account-token

---
# ClusterRoleBinding
metadata:
  name: grant-dashboard-full-access
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: dashboard-identity      # <-- must match the SA's real name
    namespace: kubernetes-dashboard
```

This works exactly the same way — the names are just labels; what matters is which **fields point to which real objects**.

## So which fields actually create the relationships?

Only three specific fields matter for wiring things together. Everything else is just a human-readable label with no functional effect on the relationship:

| Field | What it actually does |
|---|---|
| `subjects[].name` (in ClusterRoleBinding) | **Points to** the ServiceAccount that should receive permissions. This must match the ServiceAccount's real `metadata.name` + `namespace`. |
| `roleRef.name` (in ClusterRoleBinding) | **Points to** the ClusterRole whose permissions are being granted. Must match a real ClusterRole's `metadata.name` (e.g. `cluster-admin`). |
| `metadata.name` (on any object, including the ClusterRoleBinding itself) | **Only names that specific object.** It is never used by Kubernetes to "find" or "link to" anything else. |

Also worth calling out from note 5: the Secret's link to its ServiceAccount comes from its **annotation** (`kubernetes.io/service-account.name`), not from the Secret's own `metadata.name`.

```
        THIS creates the link              THIS is just a label
              |                                    |
              v                                    v
subjects:                              metadata:
  - name: admin-user   <---- points -->  name: admin-user  (ClusterRoleBinding's own name,
                                                              could be anything)
```

## Why people name them the same anyway

Even though it's not required, giving everything related the same name (`admin-user`) is a **very common human convention**, because:

- It makes it easy to see, at a glance, that these three objects are "part of the same setup."
- It avoids needing to remember a separate mapping of names.

But it's a **readability choice for humans**, not a **technical requirement for Kubernetes**. The actual relationships are always created by the specific pointer fields (`subjects[].name`, `roleRef.name`, and the Secret's annotation) — never by matching `metadata.name` values.

## Quick recap

- `metadata.name` only names the object it's attached to — it never links anything.
- `subjects[].name` in a binding is what actually points to a ServiceAccount.
- `roleRef.name` in a binding is what actually points to a ClusterRole.
- A Secret links to its ServiceAccount via the `kubernetes.io/service-account.name` annotation, not its own name.
- Same names everywhere = a helpful human convention, not a Kubernetes requirement.
