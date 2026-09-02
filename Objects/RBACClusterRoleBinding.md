# 6. RBAC / ClusterRoleBinding

## The exact YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

## Line by line

| Line | Meaning |
|---|---|
| `apiVersion: rbac.authorization.k8s.io/v1` | This object belongs to Kubernetes' **RBAC (Role-Based Access Control)** API group — the permissions system. |
| `kind: ClusterRoleBinding` | The "connector" object that links an identity to a set of permissions, cluster-wide. |
| `metadata.name: admin-user` | Just the name of this **binding object itself** — see note 7 for why this is easy to confuse with the ServiceAccount's name. |
| `roleRef` | Says **which permissions** to grant. |
| `roleRef.apiGroup: rbac.authorization.k8s.io` | The API group the referenced role belongs to. |
| `roleRef.kind: ClusterRole` | Says we're referencing a **ClusterRole** (cluster-wide permission set), not a namespace-scoped `Role`. |
| `roleRef.name: cluster-admin` | The specific ClusterRole being granted — `cluster-admin` is a **built-in** Kubernetes ClusterRole that has full power over everything. |
| `subjects` | Says **who** receives these permissions. Can be a list — you can bind multiple identities in one binding, though here there's just one. |
| `subjects[0].kind: ServiceAccount` | The identity type receiving permissions — here, a ServiceAccount (could also be a `User` or `Group`). |
| `subjects[0].name: admin-user` | The **name of the ServiceAccount** receiving the permissions. |
| `subjects[0].namespace: kubernetes-dashboard` | The namespace that ServiceAccount lives in (needed because ServiceAccount names are only unique per-namespace). |

## What is a ClusterRoleBinding?

A **ClusterRoleBinding** is the object that **connects an identity to a permission set**, applying that permission cluster-wide (across every namespace).

```
ClusterRoleBinding = the WIRE connecting WHO to WHAT-THEY-CAN-DO
```

It doesn't define permissions itself, and it doesn't define an identity itself — it just links two things that already exist elsewhere:

```
   subjects (WHO)  <----- ClusterRoleBinding -----> roleRef (WHAT PERMISSIONS)
   ServiceAccount                                    ClusterRole
    admin-user                                       cluster-admin
```

## What is a ClusterRole?

A **ClusterRole** is a **definition of permissions** — a list of rules like "can read Pods," "can create Deployments," "can delete anything in any namespace," etc. It exists **cluster-wide** (unlike a plain `Role`, which is scoped to one namespace).

`cluster-admin` is a special **built-in** ClusterRole that Kubernetes ships with by default. It effectively means: **"allowed to do absolutely anything, to any resource, in any namespace."**

## What is `roleRef`?

`roleRef` is the part of the binding that says **which** ClusterRole (or Role) is being granted. It's a pointer — "go look up the ClusterRole named `cluster-admin`, and that's the permission set I'm granting here."

## What is `subjects`?

`subjects` is the part of the binding that says **who** is receiving that permission set. It's a list because one binding can grant the same role to multiple identities at once (multiple ServiceAccounts, Users, or Groups).

## Exactly how does the ServiceAccount get the permissions?

Nothing about the ServiceAccount itself changes. Permissions are not "stored on" the ServiceAccount object. Instead:

```
Step 1: ServiceAccount "admin-user" exists (note 4) — just an identity, no powers.

Step 2: ClusterRoleBinding "admin-user" exists (this note) and says:
         "grant cluster-admin's permissions to ServiceAccount admin-user"

Step 3: Whenever a request arrives at the API Server carrying a token
         that identifies as ServiceAccount admin-user, the API Server
         checks: "is there any RoleBinding/ClusterRoleBinding that
         connects this identity to a role?"

Step 4: It finds this ClusterRoleBinding, follows roleRef to
         ClusterRole "cluster-admin", and allows the request.
```

The permission is granted **dynamically, at request time** — the API Server evaluates bindings on every request; nothing is "baked into" the ServiceAccount permanently.

## Why is `cluster-admin` extremely powerful?

Because it removes **every restriction**. A ServiceAccount bound to `cluster-admin` can:

- Read, create, modify, or delete **any resource**, in **any namespace**.
- Create or delete other ServiceAccounts, Roles, and bindings — including granting itself (or anything else) even more access.
- Access Secrets across the entire cluster (meaning it could read every password, token, and certificate stored anywhere).
- Effectively do anything a cluster super-administrator could do, including destructive things like deleting entire namespaces or workloads.

```
cluster-admin
   |
   +-- can read/write/delete ANYTHING
   +-- in EVERY namespace
   +-- including all Secrets, all workloads, all RBAC rules
```

This is why binding `cluster-admin` to a ServiceAccount (as done in this dashboard example) should be treated as a **deliberate, high-trust decision** — usually fine for a personal learning cluster or a fully trusted admin tool, but risky for anything exposed more broadly.

## Least privilege

The **principle of least privilege** means: **give an identity only the permissions it actually needs to do its job — nothing more.**

Instead of granting `cluster-admin` broadly, a more careful setup would:

- Create a **custom ClusterRole** (or namespace-scoped `Role`) listing only the specific verbs (`get`, `list`, `watch`, `create`, etc.) and resources (`pods`, `deployments`, `configmaps`, etc.) actually needed.
- Bind that narrower role instead of `cluster-admin`.

```
Bad (broad):     ServiceAccount --- cluster-admin ---> everything, everywhere
Better (narrow): ServiceAccount --- pod-reader ------> only "get/list pods" in one namespace
```

Using `cluster-admin` for the Kubernetes Dashboard is common in **learning/demo setups** specifically because it's simple and shows you the whole cluster — but in a real production environment, you'd normally scope the Dashboard's ServiceAccount down to only what it truly needs.

## Quick recap

- ClusterRoleBinding connects an identity (`subjects`) to a permission set (`roleRef`), cluster-wide.
- ClusterRole defines *what* is allowed; it grants nothing by itself until bound.
- `cluster-admin` = full, unrestricted power over the whole cluster.
- Least privilege = always prefer the smallest permission set that still gets the job done.
