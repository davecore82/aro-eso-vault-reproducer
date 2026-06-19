# Phase 3: PushSecret + CronJob Automation Solution

## Overview

This solution uses ESO's PushSecret feature combined with a Kubernetes CronJob to fully automate pull-secret management in ARO. 

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
│ CronJob runs every 1 minute                 │
│ Merges platform + customer                  │
│ Writes to Vault: final-merged               │
└──────────────┬──────────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────────┐
│ ExternalSecret pulls merged result          │
│ Syncs to pull-secret every 30 seconds       │
└──────────────┬──────────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────────┐
│ pull-secret (platform + customer)           │
└─────────────────────────────────────────────┘
```

## Benefits

✅ **Fully automated** - No manual intervention after setup  
✅ **No sync loop** - CronJob writes to separate `final-merged` path (avoids Phase 4 issue)  
✅ **Fast updates** - Customer credential changes reflected within 1-2 minutes  
✅ **ARO rotation handling** - Platform credentials auto-synced and merged  

## Complete Implementation

### Prerequisites

- HashiCorp Vault deployed in cluster
- External Secrets Operator v0.9.11+
- Vault SecretStore configured in openshift-config namespace

### Step 1: Deploy Vault SecretStore

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: openshift-config
type: Opaque
stringData:
  token: root  # Use your Vault token
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: openshift-config
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        tokenSecretRef:
          name: vault-token
          key: token
```

### Step 2: Create PushSecret for Platform Credentials

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

### Step 3: Store Customer Credentials

```bash
# Create customer credentials
cat > /tmp/customer-creds.json <<EOF
{
  "auths": {
    "mycustomer.azurecr.io": {
      "auth": "$(echo -n 'customer-user:customer-password' | base64)"
    }
  }
}
