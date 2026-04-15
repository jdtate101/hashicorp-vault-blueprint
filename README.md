# Backing Up HashiCorp Vault on Kubernetes with Kasten K10

*A Kanister Blueprint that automatically unseals Vault before a Kasten snapshot and re-seals it afterwards — solving the sealed pod readiness problem without manual intervention.*

---

HashiCorp Vault on Kubernetes runs as a StatefulSet. On pod restart or node failure, Vault starts in a sealed state — it holds encrypted data in storage but cannot serve any requests until it is unsealed with one or more unseal key shards. A sealed pod reports as `0/1` not ready, and Kasten K10 checks pod readiness before snapshotting PVCs. The result: a sealed Vault pod causes the backup job to stall or fail.

The solution is a Kanister Blueprint attached to the Kasten backup policy as pre/post hooks. The Blueprint unseals Vault before the snapshot and re-seals it afterwards, handling all three possible states (already sealed, already unsealed, or don't re-seal) gracefully.

---

## How Vault sealing works

Sealed → pod is `0/1`, Vault API is unavailable. Unsealed → pod is `1/1`, Vault API is available. Kasten needs the pod to be `1/1` before it will proceed with snapshotting the PVC.

The Blueprint workflow looks like this:

```
Kasten Backup Policy triggered
        │
        ▼
  backupPrehook fires
        │
        ├─ KubeTask pod starts in vault namespace
        ├─ Reads unseal_key from vault-unseal-key secret
        ├─ Finds vault-0 pod via label selector
        └─ Runs: vault operator unseal <key>
        │
        ▼
  Vault pod becomes 1/1 Ready
        │
        ▼
  Kasten snapshots PVCs
        │
        ▼
  backupPosthook fires
        │
        ├─ KubeTask pod starts in vault namespace
        ├─ Reads root_token from vault-unseal-key secret
        ├─ Finds vault-0 pod via label selector
        └─ Runs: vault operator seal
        │
        ▼
  Vault returns to sealed state
```

---

## Prerequisites

### 1. Store the unseal key and root token as a Kubernetes Secret

The Blueprint needs the unseal key to unseal Vault and the root token (or a token with `sudo` on `sys/seal`) to re-seal it.

```bash
oc create secret generic vault-unseal-key \
  -n vault \
  --from-literal=unseal_key="<your-unseal-key>" \
  --from-literal=root_token="<your-root-token>"
```

> The root token grants full Vault access. For production, consider creating a dedicated Vault token with a policy scoped only to `sys/seal`.

If you need to generate a new root token:

```bash
# Step 1 — Initialise root token generation
oc exec -n vault vault-0 -- vault operator generate-root -init

# Step 2 — Provide your unseal key (use nonce from step 1)
oc exec -n vault vault-0 -- vault operator generate-root \
  -nonce=<nonce> <unseal-key>

# Step 3 — Decode the encoded token (use OTP from step 1)
oc exec -n vault vault-0 -- vault operator generate-root \
  -decode=<encoded-token> \
  -otp=<otp>
```

To update the secret later (e.g. after rotating the root token):

```bash
oc create secret generic vault-unseal-key \
  -n vault \
  --from-literal=unseal_key="<key>" \
  --from-literal=root_token="<new-token>" \
  --dry-run=client -o yaml | oc apply -f -
```

### 2. Create RBAC for the Kanister job pod

Kanister spawns a temporary KubeTask pod to execute the Blueprint phases. This pod needs permission to list and exec into pods in the `vault` namespace, and to read the secret containing the unseal key and root token.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-kanister
  namespace: vault
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: vault-kanister
  namespace: vault
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: vault-kanister
  namespace: vault
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: vault-kanister
subjects:
  - kind: ServiceAccount
    name: vault-kanister
    namespace: vault
```

Apply it:

```bash
oc apply -f vault-kanister-rbac.yaml
```

> Some versions of Kanister ignore the `podOverride.serviceAccountName` field and fall back to the default service account. To cover this, also bind the role to `default`:

```bash
oc create rolebinding vault-default-kanister \
  --role=vault-kanister \
  --serviceaccount=vault:default \
  -n vault
```

---

## The Blueprint

The Blueprint defines two actions that Kasten invokes as hooks on the backup policy:

| Action | When it runs | What it does |
|---|---|---|
| `backupPrehook` | Before snapshot | Unseals Vault |
| `backupPosthook` | After snapshot | Seals Vault |

Each action spawns a KubeTask pod running `quay.io/openshift/origin-cli:latest`, which has the `oc` binary available to interact with the cluster. The Blueprint must live in the `kasten-io` namespace so that Kasten can reference it.

Save this as `vault-seal-unseal-hooks.yaml`:

```yaml
apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: vault-seal-unseal-hooks
  namespace: kasten-io
actions:
  backupPrehook:
    phases:
      - func: KubeTask
        name: unseal
        args:
          namespace: vault
          image: quay.io/openshift/origin-cli:latest
          podOverride:
            spec:
              serviceAccountName: vault-kanister
          command:
            - /bin/sh
            - -c
            - |
              VAULT_POD=$(oc get pods -n vault -l app.kubernetes.io/name=vault -o jsonpath='{.items[0].metadata.name}')
              VAULT_UNSEAL_KEY=$(oc get secret vault-unseal-key -n vault -o jsonpath='{.data.unseal_key}' | base64 --decode)
              echo "Unsealing Vault pod: $VAULT_POD"
              oc exec -n vault $VAULT_POD -- vault operator unseal $VAULT_UNSEAL_KEY
  backupPosthook:
    phases:
      - func: KubeTask
        name: seal
        args:
          namespace: vault
          image: quay.io/openshift/origin-cli:latest
          podOverride:
            spec:
              serviceAccountName: vault-kanister
          command:
            - /bin/sh
            - -c
            - |
              VAULT_POD=$(oc get pods -n vault -l app.kubernetes.io/name=vault -o jsonpath='{.items[0].metadata.name}')
              VAULT_TOKEN=$(oc get secret vault-unseal-key -n vault -o jsonpath='{.data.root_token}' | base64 --decode)
              echo "Sealing Vault pod: $VAULT_POD"
              oc exec -n vault $VAULT_POD -- env VAULT_TOKEN=$VAULT_TOKEN vault operator seal
```

Apply it:

```bash
oc apply -f vault-seal-unseal-hooks.yaml
```

---

## Attaching the Blueprint to a Kasten policy

In the Kasten K10 UI, navigate to **Policies** and edit (or create) your Vault backup policy. Under **Hooks**, set:

- Pre-snapshot hook → Blueprint: `vault-seal-unseal-hooks`, Action: `backupPrehook`
- Post-snapshot hook → Blueprint: `vault-seal-unseal-hooks`, Action: `backupPosthook`

Save and run a manual backup to verify.

---

## Seal/unseal behaviour during backup

The Blueprint handles all three states gracefully:

| Vault state at backup time | Behaviour |
|---|---|
| Sealed | Prehook unseals it → backup runs → posthook re-seals it |
| Already unsealed | `vault operator unseal` is a no-op → backup runs → posthook seals it |
| Don't want it re-sealed | Remove the posthook from the policy — prehook still unseals for the backup, Vault remains unsealed afterwards |

In development environments where you don't want to manage unsealing on every restart, configure the policy with only the pre-snapshot hook and omit the post-snapshot hook entirely.

---

## Verifying the setup

```bash
# Check current seal status
oc exec -n vault vault-0 -- vault status

# Manually unseal (for testing)
VAULT_UNSEAL_KEY=$(oc get secret vault-unseal-key -n vault \
  -o jsonpath='{.data.unseal_key}' | base64 --decode)
oc exec -n vault vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY

# Manually seal (for testing)
VAULT_TOKEN=$(oc get secret vault-unseal-key -n vault \
  -o jsonpath='{.data.root_token}' | base64 --decode)
oc exec -n vault vault-0 -- env VAULT_TOKEN=$VAULT_TOKEN vault operator seal
```
