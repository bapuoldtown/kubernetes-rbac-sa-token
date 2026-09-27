# Stage 3 — RBAC & Identity

**Concept.** Every pod runs *as* an identity: a **ServiceAccount (SA)** — its badge.
Pick none and the pod silently gets the namespace's `default` SA, whose **token is
auto-mounted inside the pod** at `/var/run/secrets/kubernetes.io/serviceaccount/token`.
That token is a real login credential to the Kubernetes API server. **RBAC** (Roles +
bindings) decides what that badge is allowed to do — and it denies everything until a
Role explicitly grants it.

Two default problems this stage fixes:
1. The credential sits in *every* pod, even apps that never call the API — free loot for
   an attacker who lands in the pod.
2. People over-grant the `default` SA out of laziness.

> Note: a ServiceAccount is only for talking to the **Kubernetes API server**. Containers
> in the same pod talking to each other (e.g. app → its DB on `localhost:5432`) is plain
> TCP — no badge, no RBAC involved. And the badge is **per pod**, not per container.

## How a request is authorized

When a pod calls the API (`kubectl auth can-i list pods` from inside it):

```
inner kubectl → reads token file → HTTPS to API server with token attached
      │
      ▼
API SERVER
  [1] AUTHENTICATION — "who are you?"   verifies the token signature
      → system:serviceaccount:s3-rbac:demo-sa   (bad token → 401)
  [2] AUTHORIZATION (RBAC) — "are you allowed?"   scans Roles bound to demo-sa
      → no matching rule → deny ("no") ;  matching rule → allow ("yes")
```

Two separate checkpoints:

| Step | Question | Decided by | Fails with |
|---|---|---|---|
| Authentication | Who are you? | the **token** (signature) | `401 Unauthorized` |
| Authorization | Are you allowed? | **RBAC** (Roles + bindings) | `403` / `no` |

Authentication always succeeds here (the token is real); what changes is authorization.

## Step 1 — a badge and a pod that wears it

`s3-sa-pod.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-sa
  namespace: s3-rbac
---
apiVersion: v1
kind: Pod
metadata:
  name: probe
  namespace: s3-rbac
spec:
  serviceAccountName: demo-sa        # wear THIS badge, not the default one
  containers:
    - name: app
      image: bitnami/kubectl:latest  # kubectl inside, so we can swipe the badge
      command: ["sleep", "3600"]
```

```powershell
kubectl create ns s3-rbac
kubectl apply -f s3-sa-pod.yaml
kubectl wait --for=condition=Ready pod/probe -n s3-rbac --timeout=90s

kubectl exec -n s3-rbac probe -- kubectl auth can-i list pods   # -> no (deny by default)
```

A brand-new badge opens zero doors.

## Step 2 — grant exactly one door

- **Role** = a keycard listing which actions on which resources.
- **RoleBinding** = giving that keycard to a badge.

`s3-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: s3-rbac
rules:
  - apiGroups: [""]              # "" = core API group (pods live here)
    resources: ["pods"]
    verbs: ["get", "list"]      # ONLY read
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-sa-can-read-pods
  namespace: s3-rbac
subjects:
  - kind: ServiceAccount
    name: demo-sa
    namespace: s3-rbac
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```powershell
kubectl apply -f s3-role.yaml
kubectl exec -n s3-rbac probe -- kubectl auth can-i list pods     # -> yes
kubectl exec -n s3-rbac probe -- kubectl auth can-i delete pods   # -> no  (never granted)
kubectl exec -n s3-rbac probe -- kubectl get pods -n s3-rbac      # actually works now
```

`list = yes` but `delete = no` **is** least privilege: one door open, the rest shut.

## The prod move: remove the badge entirely

Most apps (a web app + its DB) call the Kubernetes API **zero** times, so the safest
thing is to carry no credential at all. Set `automountServiceAccountToken: false` and an
attacker who breaks in finds no token to steal.

`s3-no-badge.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-badge
  namespace: s3-rbac
spec:
  automountServiceAccountToken: false     # no token mounted into the pod
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
```

```powershell
kubectl apply -f s3-no-badge.yaml
kubectl wait --for=condition=Ready pod/no-badge -n s3-rbac --timeout=60s
kubectl exec -n s3-rbac no-badge -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# ls: .../serviceaccount/: No such file or directory   <- the badge is gone
```

You can set this on the ServiceAccount (applies to all its pods) or per-pod (overrides
the SA). Rule of thumb: default it off, opt in only for the rare pod that manages
Kubernetes itself (a controller, operator, or CI runner).

## Cleanup

```powershell
kubectl delete ns s3-rbac
```