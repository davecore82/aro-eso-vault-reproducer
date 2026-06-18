# Phase 3: PushSecret Solution 

## Overview

This phase demonstrates a solution using ESO's `PushSecret` feature to automatically sync ARO's platform credentials to Vault. This eliminates the manual monitoring requirement from Phase 2.

## Architecture

```
┌─────────────────────────────────────────────┐
│ ARO updates pull-secret                     │
│ (platform credential rotation)              │
└──────────────┬──────────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────────┐
│ PushSecret watches pull-secret              │
│ Auto-syncs TO Vault every 30 seconds        │
└──────────────┬──────────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────────┐
│ Vault: platform-pull-secret-auto            │
│ (Always current with ARO credentials)       │
└─────────────────────────────────────────────┘
               +
┌─────────────────────────────────────────────┐
│ Vault: customer-only-registry               │
│ (Customer manages via ESO/automation)       │
└──────────────┬──────────────────────────────┘
               │
               ↓ (merge in Vault)
               │
┌─────────────────────────────────────────────┐
│ Vault: final-merged                         │
│ (Platform + Customer combined)              │
└──────────────┬──────────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────────┐
│ ExternalSecret pulls merged result          │
│ Syncs back to pull-secret                   │
└─────────────────────────────────────────────┘
```

## Benefits

✅ **No manual monitoring** - ARO updates are automatically pushed to Vault  
✅ **Credential separation** - Customer credentials stay separate from platform credentials  
✅ **Always in sync** - PushSecret keeps Vault current with ARO's rotations  
✅ **Scalable** - Works for any number of customer registries  

## Implementation

### Step 1: Create PushSecret to Auto-Sync Platform Credentials

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: push-platform-credentials
  namespace: openshift-config
spec:
  refreshInterval: 30s
  secretStoreRefs:
  - name: vault-backend
    kind: SecretStore
  selector:
    secret:
      name: pull-secret
  data:
  - match:
      secretKey: .dockerconfigjson
      remoteRef:
        remoteKey: platform-pull-secret-auto
        property: .dockerconfigjson
```

Apply:
```bash
oc apply -f pushsecret.yaml
```

Verify it's syncing:
```bash
oc get pushsecret push-platform-credentials -n openshift-config
# Should show: STATUS: Synced

# Check Vault
oc exec -n vault deployment/vault -- vault kv get secret/platform-pull-secret-auto
```

### Step 2: Store Customer Credentials Separately

```bash
# Create customer-only credentials
cat > /tmp/customer-creds.json <<EOF
{
  "auths": {
    "mycustomer.azurecr.io": {
      "auth": "$(echo -n 'customer-user:customer-password' | base64)"
    }
  }
}
EOF

# Store in Vault at separate path
oc exec -n vault deployment/vault -- vault kv put secret/customer-only-registry \
  .dockerconfigjson="$(cat /tmp/customer-creds.json)"
```

### Step 3: Merge Credentials in Vault

This is the remaining challenge - automating the merge. Options:

#### Option A: Manual Merge (Simple)

```bash
# Read platform credentials (auto-updated by PushSecret)
PLATFORM=$(oc exec -n vault deployment/vault -- \
  vault kv get -format=json secret/platform-pull-secret-auto | \
  jq -r '.data.data[".dockerconfigjson"]')

# Read customer credentials
CUSTOMER=$(oc exec -n vault deployment/vault -- \
  vault kv get -format=json secret/customer-only-registry | \
  jq -r '.data.data[".dockerconfigjson"]')

# Merge (platform takes precedence on conflicts)
MERGED=$(echo "$PLATFORM" | jq --argjson customer "$CUSTOMER" '.auths += $customer.auths')

# Store merged result
echo "$MERGED" | oc exec -i -n vault deployment/vault -- \
  vault kv put secret/final-merged .dockerconfigjson=-

# Verify
oc exec -n vault deployment/vault -- \
  vault kv get -format=json secret/final-merged | \
  jq -r '.data.data[".dockerconfigjson"]' | jq '.auths | keys'
```

#### Option B: Automated Merge (Advanced)

Create a sidecar job or CronJob that:
1. Watches both Vault paths (`platform-pull-secret-auto` and `customer-only-registry`)
2. Automatically merges when either changes
3. Writes to `final-merged`

#### Option C: Vault Policy/Transform (Enterprise)

Use Vault's transformation capabilities (requires Vault Enterprise) to merge automatically.

### Step 4: ExternalSecret Pulls Merged Result

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
      key: final-merged
```

Apply:
```bash
oc apply -f externalsecret-merged.yaml
```

## Testing the Solution

### Test 1: Verify PushSecret Auto-Sync

```bash
# Check PushSecret status
oc get pushsecret push-platform-credentials -n openshift-config

# Verify platform credentials in Vault
oc exec -n vault deployment/vault -- \
  vault kv get -format=json secret/platform-pull-secret-auto | \
  jq -r '.data.data[".dockerconfigjson"]' | jq '.auths | keys'

# Should show all 6 ARO registries:
# - arosvc.azurecr.io
# - quay.io
# - registry.redhat.io
# - registry.connect.redhat.com
# - cloud.openshift.com
# - sso.redhat.com
```

### Test 2: Verify Merge Works

```bash
# Check merged credentials have both platform + customer
oc exec -n vault deployment/vault -- \
  vault kv get -format=json secret/final-merged | \
  jq -r '.data.data[".dockerconfigjson"]' | jq '.auths | keys'

# Should show 7 registries (6 ARO + 1 customer)
```

### Test 3: Simulate ARO Credential Rotation

```bash
# Simulate ARO updating pull-secret
oc get secret pull-secret -n openshift-config -o json | \
  jq '.data[".dockerconfigjson"] |= "FAKE-UPDATE"' | \
  oc apply -f -

# Wait 35 seconds for PushSecret to sync
sleep 35

# Check PushSecret pushed the update to Vault
oc exec -n vault deployment/vault -- \
  vault kv get secret/platform-pull-secret-auto
```

## Comparison with Phase 2

| Aspect | Phase 2 (Manual) | Phase 3 (PushSecret) |
|--------|------------------|----------------------|
| **Platform credential sync** | Manual copy/update | ✅ Automatic via PushSecret |
| **ARO rotation handling** | ⚠️ Requires monitoring | ✅ Auto-synced to Vault |
| **Customer credential updates** | Via ESO or manual | Via ESO or manual |
| **Merge complexity** | Done once upfront | Need automation |
| **Risk of drift** | ⚠️ High if not monitored | ✅ Low - always in sync |

## Remaining Work

The **merge step** needs automation. Currently this is the only manual step. Options:

1. **Simple**: Run merge script as CronJob (every minute)
2. **Better**: Create sidecar that watches Vault KV and merges on change
3. **Best**: Use Vault Enterprise transforms or ESO template features

## HyperShift Reference

This solution is inspired by HyperShift's approach to pull-secret management:
- **HyperShift**: Uses controller to merge `original-pull-secret` + `additional-pull-secret`
- **This solution**: Uses PushSecret + Vault merge + ExternalSecret
- **Key difference**: HyperShift merges in OpenShift controller; we merge in Vault

See HyperShift implementation:
- Controller: `control-plane-operator/hostedclusterconfigoperator/controllers/globalps/globalps.go`
- Sync daemon: `sync-global-pullsecret/sync-global-pullsecret.go`

## Conclusion

PushSecret solves the **manual monitoring problem** from Phase 2. The only remaining challenge is automating the merge step in Vault, which can be addressed with:
- Simple CronJob (good enough for most use cases)
- Vault Enterprise features (if available)
- Custom operator/controller (if needed at scale)

