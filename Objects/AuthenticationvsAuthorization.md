# 8. Authentication vs Authorization

Kubernetes access control always splits into two separate questions, handled by two completely separate systems:

1. **Authentication** — "Who are you?"
2. **Authorization** — "What are you allowed to do?"

## The simple mental model

```
ServiceAccount   =   WHO           (an identity)
Token            =   PROOF         (a credential that proves the identity)
RBAC             =   WHAT YOU CAN DO   (permissions checked after identity is confirmed)
```

Keep these three roles distinct in your head — a huge amount of RBAC confusion comes from mixing them up:

- The **ServiceAccount** never proves anything by itself. It's just a name.
- The **Token** never grants any permission by itself. It only proves identity.
- **RBAC** (ClusterRole + ClusterRoleBinding) never proves identity. It only decides what an *already confirmed* identity can do.

## The complete flow, step by step

```
 ServiceAccount (admin-user)
        |
        |  has a token generated for it (note 5)
        v
     Token
        |
        |  sent in a request:  Authorization: Bearer <token>
        v
 Kubernetes API Server
        |
        |  STEP 1 - AUTHENTICATION
        |  verifies the token's signature
        |  -> "this request really is from admin-user"
        v
   Identity recognized
        |
        |  STEP 2 - AUTHORIZATION (RBAC)
        |  looks up: is there a RoleBinding/ClusterRoleBinding
        |  connecting admin-user to some Role/ClusterRole?
        |  -> finds ClusterRoleBinding "admin-user" -> cluster-admin
        v
   Request allowed or denied
        |
        +-- ALLOWED  -> action performed (e.g. list all Pods)
        +-- DENIED   -> "Forbidden" error returned
```

## Why this two-step design matters

Splitting authentication and authorization means:

- You could have a perfectly **valid, correctly signed token** for a ServiceAccount that has **zero permissions** — authentication succeeds, but authorization fails, and every request gets a "Forbidden" response.
- You could (in theory) recognize an identity without any binding existing at all — Kubernetes would authenticate the request just fine, but then reject every action because no ClusterRoleBinding/RoleBinding connects that identity to any permissions.

This is exactly why setting up access for something like the Dashboard needs **all three pieces** — a ServiceAccount (WHO), a Secret with a token (PROOF), and a ClusterRoleBinding (WHAT) — miss any one of them and the whole thing breaks, but for different reasons:

| Missing piece | What breaks |
|---|---|
| No ServiceAccount | Nothing to authenticate as in the first place. |
| No Token/Secret | No way to prove identity — authentication fails outright. |
| No ClusterRoleBinding | Identity is confirmed, but every action is denied ("Forbidden"). |

## Quick recap

- Authentication answers "who is making this request?" — handled by the token.
- Authorization answers "is this identity allowed to do this?" — handled by RBAC (ClusterRole + ClusterRoleBinding).
- These are always two separate checks, in that order — identity first, permission second.
