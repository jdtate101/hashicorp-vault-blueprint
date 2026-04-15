# hashicorp-vault-blueprint
Overview
HashiCorp Vault on Kubernetes runs as a StatefulSet. When Vault is sealed, its pods report as 0/1 not ready — which causes Kasten K10 to either skip the workload or flag the backup as unhealthy, since it expects all pods to be in a running and ready state before snapshotting.
The solution is to use a Kanister Blueprint as a pre/post hook on the Kasten backup policy. The Blueprint automatically unseals Vault before the snapshot is taken, and optionally re-seals it afterwards.---


How Vault Works on Kubernetes
Vault is typically deployed via the official Helm chart as a StatefulSet (vault-0, vault-1, etc.). On pod restart or node failure, Vault starts in a sealed state — it holds encrypted data in storage but cannot serve any requests until it is unsealed with one or more unseal key shards.
This means:
Sealed → pod is 0/1, Vault API is unavailable
Unsealed → pod is 1/1, Vault API is available
Kasten K10 checks pod readiness before snapshotting PVCs. A sealed Vault pod will cause the backup job to stall or fail unless the pod is unsealed first.---


Prerequisites
1. Store the Unseal Key and Root Token as a Kubernetes Secret
The Blueprint needs the unseal key to unseal Vault, and the root token (or a token with sudo on sys/seal) to re-seal it.oc create secret generic vault-unseal-key \
  -n vault \
  --from-literal=unseal_key="<your-unseal-key>" \
  --from-literal=root_token="<your-root-token>"



Security note: The root token grants full Vault access. Consider creating a dedicated Vault token with a policy scoped only to sys/seal for production use.
If you need to generate a new root token, use the generate-root workflow:# Step 1 - Initialise root token generation
oc exec -n vault vault-0 -- vault operator generate-root -init

# Step 2 - Provide your unseal key (use nonce from step 1)
oc exec -n vault vault-0 -- vault operator generate-root \
  -nonce=<nonce> <unseal-key>

# Step 3 - Decode the encoded token (use OTP from step 1)
oc exec -n vault vault-0 -- vault operator generate-root \
  -decode=<encoded-token> \
  -otp=<otp>



To update the secret later (e.g. after rotating the root token):oc create secret generic vault-unseal-key \
  -n vault \
  --from-literal=unseal_key="<key>" \
  --from-literal=root_token="<new-token>" \
  --dry-run=client -o yaml | oc apply -f -


---


2. Create RBAC for the Kanister Job Pod
Kanister spawns a temporary pod (a KubeTask) to execute the Blueprint phases. This pod needs permission to:
List pods in the vault namespace (to find vault-0)
Exec into pods in the vault namespace (to run vault operator unseal/seal)
Read secrets in the vault namespace (to retrieve the unseal key and root token)
Create a dedicated ServiceAccount, Role, and RoleBinding:apiVersion: v1
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



Apply it:oc apply -f vault-kanister-rbac.yaml



Note: Some versions of Kanister ignore the podOverride.serviceAccountName field and fall back to the default service account in the target namespace. To cover this, also bind the role to the default SA:oc create rolebinding vault-default-kanister \
  --role=vault-kanister \
  --serviceaccount=vault:default \
  -n vault


---


The Kanister Blueprint
The Blueprint defines two actions that Kasten K10 invokes as hooks on the backup policy:<table>
<thead>
<tr>
<th>Action</th>
<th>When it runs</th>
<th>What it does</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>backupPrehook</code></td>
<td>Before snapshot</td>
<td>Unseals Vault</td>
</tr>
<tr>
<td><code>backupPosthook</code></td>
<td>After snapshot</td>
<td>Seals Vault</td>
</tr>
</tbody>
</table>


Each action spawns a temporary KubeTask pod running quay.io/openshift/origin-cli:latest, which has the oc binary available to interact with the cluster.
Blueprint WorkflowKasten Backup Policy triggered
        │
        ▼
  backupPrehook fires
        │
        ├─ KubeTask pod starts in vault namespace
        ├─ Reads unseal_key from vault-unseal-key secret
        ├─ Finds vault-0 pod name via label selector
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
        ├─ Finds vault-0 pod name via label selector
        └─ Runs: vault operator seal
        │
        ▼
  Vault returns to sealed state



Blueprint YAML
Save this as vault-seal-unseal-hooks.yaml. The Blueprint must live in the kasten-io namespace so that Kasten can reference it.apiVersion: cr.kanister.io/v1alpha1
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



Apply it:oc apply -f vault-seal-unseal-hooks.yaml


---


Attaching the Blueprint to a Kasten Policy
In the Kasten K10 UI:
Navigate to Policies and edit (or create) your Vault backup policy
Under Hooks, set:
Pre-snapshot hook → Blueprint: vault-seal-unseal-hooks, Action: backupPrehook
Post-snapshot hook → Blueprint: vault-seal-unseal-hooks, Action: backupPosthook
Save and run a manual backup to verify---


Seal/Unseal Behaviour During Backup
The Blueprint handles all three states gracefully:<table>
<thead>
<tr>
<th>Vault State at Backup Time</th>
<th>Behaviour</th>
</tr>
</thead>
<tbody>
<tr>
<td><strong>Sealed</strong></td>
<td>Prehook unseals it → backup runs → posthook re-seals it</td>
</tr>
<tr>
<td><strong>Already unsealed</strong></td>
<td><code>vault operator unseal</code> is a no-op if already unsealed → backup runs → posthook seals it</td>
</tr>
<tr>
<td><strong>You don't want it re-sealed</strong></td>
<td>Remove the posthook from the Kasten policy — the prehook will still unseal it for the backup but it will remain unsealed afterwards</td>
</tr>
</tbody>
</table>


Tip: If you prefer Vault to remain unsealed after backups (e.g. in a development environment where you don't want to manage unsealing on every restart), simply configure the policy with only the pre-snapshot hook and omit the post-snapshot hook entirely.---


Verifying the Setup
Check Vault status manually:# Check current seal status
oc exec -n vault vault-0 -- vault status

# Manually unseal (for testing)
VAULT_UNSEAL_KEY=$(oc get secret vault-unseal-key -n vault -o jsonpath='{.data.unseal_key}' | base64 --decode)
oc exec -n vault vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY

# Manually seal (for testing)
VAULT_TOKEN=$(oc get secret vault-unseal-key -n vault -o jsonpath='{.data.root_token}' | base64 --decode)
oc exec -n vault vault-0 -- env VAULT_TOKEN=$VAULT_TOKEN vault operator seal



