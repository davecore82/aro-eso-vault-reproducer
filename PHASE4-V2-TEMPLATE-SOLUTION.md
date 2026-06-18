# Phase 4: Template Engine v2 Solution 

## Overview

This phase demonstrates another solution using ESO's template engine v2 to merge credentials directly in the ExternalSecret. This eliminates the manual merge step from Phase 3.

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
│ (Customer manages their credentials)        │
└──────────────┬──────────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────────┐
│ ExternalSecret with v2 template             │
│ Pulls BOTH + merges with Go templates       │
└──────────────┬──────────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────────┐
│ pull-secret (merged result)                 │
└─────────────────────────────────────────────┘
```

## Benefits

✅ **Fully automated** - No manual steps after initial setup  
✅ **Native ESO merge** - Uses template engine, no external scripts  
✅ **Auto-sync** - ARO rotations automatically propagated via PushSecret  
✅ **Credential separation** - Platform and customer credentials stored separately  
✅ **Production ready** - No CronJobs, sidecars, or custom operators needed  

## Requirements

- External Secrets Operator v0.9.11+ (supports `engineVersion: v2`)
- HashiCorp Vault (or any ESO-supported secret store)

## Implementation

### Step 1: PushSecret Auto-Syncs Platform Credentials

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

Verify:
```bash
oc get pushsecret push-platform-credentials -n openshift-config
# Should show: STATUS: Synced
```

### Step 2: Store Customer Credentials Separately

```bash
# Customer credentials only
cat > /tmp/customer-creds.json <<EOF
{
  "auths": {
    "mycustomer.azurecr.io": {
      "auth": "$(echo -n 'customer-user:customer-password' | base64)"
    }
  }
}
EOF

# Store in Vault
oc exec -i -n vault deployment/vault -- \
  vault kv put secret/customer-only-registry \
  .dockerconfigjson=- < /tmp/customer-creds.json
```

### Step 3: ExternalSecret with v2 Template Merges Both

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: pull-secret-merged
  namespace: openshift-config
spec:
  refreshInterval: 30s
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: pull-secret
    creationPolicy: Merge
    template:
      engineVersion: v2
      type: kubernetes.io/dockerconfigjson
      data:
        .dockerconfigjson: |
          {{- $platform := .platform | fromJson -}}
          {{- $customer := .customer | fromJson -}}
          {{- $merged := merge $platform.auths $customer.auths -}}
          {{- dict "auths" $merged | toJson -}}
  data:
  - secretKey: platform
    remoteRef:
      key: platform-pull-secret-auto
      property: .dockerconfigjson
  - secretKey: customer
    remoteRef:
      key: customer-only-registry
      property: .dockerconfigjson
```

Apply:
```bash
oc apply -f externalsecret-v2-template.yaml
```

### How the Template Works

The template uses ESO v2's Go template engine with sprig functions:

1. **`fromJson`** - Parses JSON strings from Vault into objects
2. **`merge`** - Merges two maps (platform.auths + customer.auths)
3. **`dict`** - Creates a new map with "auths" key
4. **`toJson`** - Converts back to JSON for the dockerconfigjson format

The merge gives **platform credentials precedence** on conflicts (same behavior as HyperShift).

## Testing

### Verify Initial State

```bash
# Check ExternalSecret status
oc get externalsecret pull-secret-merged -n openshift-config
# Should show: STATUS: SecretSynced, READY: True

# Check pull-secret has merged credentials
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | \
  base64 -d | jq '.auths | keys'

# Should show all platform + customer registries:
# - arosvc.azurecr.io (ARO)
# - quay.io (Red Hat)
# - registry.redhat.io (Red Hat)
# - registry.connect.redhat.com (Red Hat)
# - mycustomer.azurecr.io (Customer)
```

### Test 1: Customer Updates Their Credentials

```bash
# Add another customer registry
cat > /tmp/customer-multi.json <<EOF
{
  "auths": {
    "mycustomer.azurecr.io": {
      "auth": "$(echo -n 'customer-user:new-password' | base64)"
    },
    "anothercustomer.azurecr.io": {
      "auth": "$(echo -n 'another-user:password' | base64)"
    }
  }
}
EOF

# Update Vault
oc exec -i -n vault deployment/vault -- \
  vault kv put secret/customer-only-registry \
  .dockerconfigjson=- < /tmp/customer-multi.json

# Wait for sync (30 seconds)
sleep 35

# Verify pull-secret updated
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | \
  base64 -d | jq '.auths | keys'

# Should now show both customer registries
```

### Test 2: ARO Rotates Platform Credentials

```bash
# Simulate ARO updating pull-secret
# (In production, ARO does this automatically)
oc get secret pull-secret -n openshift-config -o json | \
  jq '.data[".dockerconfigjson"] = "SIMULATED-UPDATE"' | \
  oc apply -f -

# PushSecret detects change and syncs to Vault (within 30s)
sleep 35

# ExternalSecret pulls updated platform creds + merges with customer creds
sleep 35

# Verify merge still works with updated platform credentials
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | \
  base64 -d | jq '.auths | keys'

# Should still have all registries (platform + customer)
```

### Test 3: Test Image Pull

```bash
# Test Red Hat registry (should work)
oc run test-redhat --image=registry.redhat.io/ubi9/ubi-minimal:latest
oc wait --for=condition=Ready pod/test-redhat --timeout=60s
oc delete pod test-redhat

# Test customer registry (replace with real image)
oc run test-customer --image=mycustomer.azurecr.io/myapp:latest
oc get pod test-customer
# Should pull successfully (or at least authenticate)
```

## Comparison with Previous Phases

| Aspect | Phase 2 | Phase 3 | Phase 4 (This) |
|--------|---------|---------|----------------|
| **Platform credential sync** | ❌ Manual | ✅ PushSecret | ✅ PushSecret |
| **Merge credentials** | ❌ Manual in Vault | ⚠️ Need automation | ✅ Template v2 |
| **Customer updates** | Manual | Via ESO/manual | Via ESO/manual |
| **ARO rotation handling** | ❌ Manual monitoring | ✅ Auto-synced | ✅ Auto-synced |
| **Complexity** | Medium | High (need merge job) | ✅ Low |
| **Production ready** | ❌ No | ⚠️ Needs merge automation | ✅ Yes |

## How It Compares to HyperShift

HyperShift uses a similar pattern but implemented differently:

### HyperShift Approach
```
original-pull-secret (platform)
        +
additional-pull-secret (user)
        ↓
Controller merges in Go code
        ↓
DaemonSet updates nodes
```

### This Solution (ESO v2)
```
platform-pull-secret-auto (via PushSecret)
        +
customer-only-registry (user managed)
        ↓
ExternalSecret template merges
        ↓
pull-secret updated
```

**Key differences:**
- HyperShift: Merge happens in custom controller (Go code)
- This solution: Merge happens in ESO template (declarative YAML)
- HyperShift: Additional-pull-secret is in the cluster
- This solution: Customer credentials in external Vault

**Same principles:**
- Platform credentials take precedence on conflicts
- Credentials stay separated until merge point
- Platform can rotate independently

## Template Engine Details

ESO v2 template engine provides:
- **200+ sprig functions** - Standard Go template helpers
- **Custom helpers** - `fromJson`, `toJson`, `fromYaml`, `toYaml`
- **Type safety** - Template errors fail the sync (vs silently corrupting data)

The merge template:
```yaml
{{- $platform := .platform | fromJson -}}
{{- $customer := .customer | fromJson -}}
{{- $merged := merge $platform.auths $customer.auths -}}
{{- dict "auths" $merged | toJson -}}
```

Is equivalent to this Go code:
```go
platform := parseJSON(.platform)
customer := parseJSON(.customer)
merged := maps.Merge(platform.auths, customer.auths)
return json.Marshal(map[string]interface{}{"auths": merged})
```

## Production Considerations

### Multiple Customer Registries

The customer can store multiple registries in one secret:
```json
{
  "auths": {
    "customer1.azurecr.io": { "auth": "..." },
    "customer2.azurecr.io": { "auth": "..." },
    "gcr.io": { "auth": "..." }
  }
}
```

All are merged into the final pull-secret.

### Credential Rotation

**Customer credentials:**
- Update Vault secret
- ESO detects change within refreshInterval (30s)
- pull-secret updated automatically

**Platform credentials:**
- ARO rotates automatically
- PushSecret detects change within refreshInterval (30s)
- ESO pulls updated platform creds and re-merges
- pull-secret updated automatically

### Monitoring

Monitor these resources:
```bash
# PushSecret syncing platform creds
oc get pushsecret push-platform-credentials -n openshift-config

# ExternalSecret syncing merged result
oc get externalsecret pull-secret-merged -n openshift-config

# Final pull-secret state
oc get secret pull-secret -n openshift-config
```

All should show healthy status.

### Conflict Resolution

If customer tries to add a registry that conflicts with platform (e.g., `arosvc.azurecr.io`):
- The `merge` function gives platform credentials precedence
- Customer's conflicting entry is ignored
- No error - silently uses platform credential

This is **by design** to prevent customers from breaking platform operations.

## Troubleshooting

### ExternalSecret not syncing

```bash
# Check ExternalSecret status
oc describe externalsecret pull-secret-merged -n openshift-config

# Check controller logs
oc logs -n external-secrets-operator deployment/external-secrets-controller
```

### Template errors

```bash
# Template syntax errors appear in ExternalSecret status
oc get externalsecret pull-secret-merged -n openshift-config -o yaml | grep -A 10 status

# Common issues:
# - Missing |- for multi-line strings
# - Missing {{- -}} for whitespace control
# - Incorrect function names (fromJson not fromJSON)
```

### PushSecret not syncing

```bash
# Check PushSecret status
oc describe pushsecret push-platform-credentials -n openshift-config

# Verify secret exists
oc get secret pull-secret -n openshift-config
```

## Conclusion

It combines:
- ✅ PushSecret (Phase 3) for automatic platform credential sync
- ✅ Template v2 for native merge in ESO
- ✅ No external scripts, CronJobs, or custom operators
- ✅ Fully declarative YAML
- ✅ Works with ESO v0.9.11+ (already widely deployed)

This solution eliminates the "merge automation" gap from Phase 3.
