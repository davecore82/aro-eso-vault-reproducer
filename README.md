# ARO Pull-Secret with ESO and Vault - Real Reproducer

Demonstrates the pull-secret management challenge in Azure Red Hat OpenShift when using External Secrets Operator with HashiCorp Vault.

## Setup

**Environment:**
- Azure Red Hat OpenShift 4.20.15
- HashiCorp Vault 1.15 (running in cluster)
- External Secrets Operator v0.9.11

**Initial State:**

ARO cluster has pull-secret with 4 platform registries:
```
- arosvc.azurecr.io (ARO platform)
- quay.io (Red Hat)
- registry.redhat.io (Red Hat)
- registry.connect.redhat.com (Red Hat)
```

## Phase 1: The Problem

Customer wants to add their private ACR registry and automate credential rotation via ESO.

### Step 1: Store customer credentials in Vault

```bash
# Customer stores only their registry in Vault
vault kv put secret/customer-registry \
  .dockerconfigjson='{
    "auths": {
      "customerregistry.azurecr.io": {
        "auth": "Y3VzdG9tZXI6cGFzc3dvcmQ=",
        "email": "ops@customer.com"
      }
    }
  }'
```

### Step 2: Configure ESO to sync to pull-secret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: pull-secret-from-vault
  namespace: openshift-config
spec:
  refreshInterval: 30s
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: pull-secret
    creationPolicy: Merge
  dataFrom:
  - extract:
      key: customer-registry
```

### Result: Platform credentials deleted

```
Before ESO sync: 4 registries
- arosvc.azurecr.io
- quay.io
- registry.redhat.io
- registry.connect.redhat.com

After ESO sync: 2 registries
- arosvc.azurecr.io (auto-restored by ARO)
- customerregistry.azurecr.io (from Vault)

DELETED:
- quay.io
- registry.redhat.io
- registry.connect.redhat.com
```

### Impact

Image pulls from Red Hat registries fail:

```
$ oc run test --image=registry.redhat.io/ubi9/ubi-minimal:latest
pod/test created

$ oc get pod test
NAME   READY   STATUS             RESTARTS   AGE
test   0/1     ImagePullBackOff   0          15s

Error: unable to retrieve auth token: invalid username/password: unauthorized
```

## Phase 2: The Workaround (Works But Fragile)

### Step 1: Store COMBINED credentials in Vault

Customer manually combines platform + customer credentials:

```bash
# Get current platform credentials from cluster
oc get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | \
  base64 -d > platform-creds.json

# Add customer registry
jq '.auths["customerregistry.azurecr.io"] = {
  "auth": "Y3VzdG9tZXI6cGFzc3dvcmQ=",
  "email": "ops@customer.com"
}' platform-creds.json > combined-creds.json

# Store combined in Vault
vault kv put secret/pull-secret-combined \
  .dockerconfigjson="$(cat combined-creds.json | jq -c .)"
```

### Step 2: Update ESO to use combined credentials

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: pull-secret-from-vault
  namespace: openshift-config
spec:
  refreshInterval: 30s
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: pull-secret
    creationPolicy: Merge
  dataFrom:
  - extract:
      key: pull-secret-combined  # Now has ALL credentials
```

### Result: Everything works!

```
Vault contains:
- arosvc.azurecr.io
- quay.io
- registry.redhat.io
- registry.connect.redhat.com
- customerregistry.azurecr.io

pull-secret contains: All 5 registries ✅
Image pulls: Working ✅
ESO: Syncing every 30 seconds ✅
```

### The Problem: This is Fragile

**When ARO rotates arosvc credentials:**

1. ARO updates `pull-secret` with new `arosvc.azurecr.io` token
2. Vault still has OLD token
3. ESO syncs from Vault (30 seconds later)
4. ESO overwrites with stale credentials
5. ARO platform operations fail

**Customer must:**
- Monitor for ARO credential rotations
- Extract new credentials
- Manually update Vault
- Repeat every time ARO rotates

**This defeats the purpose of automation!**

## Why ARO Auto-Restores arosvc.azurecr.io

ARO's config operator watches `pull-secret` and automatically restores `arosvc.azurecr.io` credentials because they're critical for platform operations.

However, it does NOT restore Red Hat registries (`quay.io`, `registry.redhat.io`, `registry.connect.redhat.com`) - those come from install-config, not runtime management.

## Related

- JIRA: ARO-25475
- Platform: Azure Red Hat OpenShift
- Test Date: June 2026
