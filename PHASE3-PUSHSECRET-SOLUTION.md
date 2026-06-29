# Phase 3: PushSecret + CronJob Automation Solution 

## Overview

This solution uses ESO's PushSecret feature combined with a Kubernetes CronJob to fully automate pull-secret management in ARO. It eliminates manual monitoring and maintenance.

**Note**: Is there a potential race condition here? 
- ARO rotates at T=0
- ExternalSecret might run at T=15s, overwriting with stale creds
- PushSecret runs at T=30s, capturing... the stale creds that ExternalSecret just wrote

With both PushSecret and ExternalSecret on 30s intervals, do we risk ExternalSecret overwriting ARO's rotation before PushSecret captures it? Should we have different intervals (PushSecret 1min, ExternalSecret 5min) to give PushSecret time to capture and propagate the rotation before ExternalSecret overwrites?

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
EOF

# Store in Vault
export VAULT_ADDR=http://vault.vault.svc.cluster.local:8200
export VAULT_TOKEN=root
vault kv put secret/customer-only-registry .dockerconfigjson=@/tmp/customer-creds.json
```

### Step 4: Deploy CronJob for Automated Merging

The CronJob merges platform and customer credentials every minute and writes the result to Vault.

**Create ServiceAccount and RBAC:**

```bash
oc create sa vault-merge -n vault
oc adm policy add-scc-to-user anyuid -z vault-merge -n vault
```

**Why anyuid SCC is required:**
- The CronJob uses `registry.access.redhat.com/ubi9/ubi-minimal:latest`
- It installs packages at runtime using `microdnf install` (jq, wget, unzip, vault)
- Package installation requires root privileges (`runAsUser: 0`)
- This is standard practice for UBI-based automation containers
- Tested with full UBI9 image - same requirement (cannot be avoided)

**Create Secret with Vault Token:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: vault
type: Opaque
stringData:
  token: root  # Use your Vault token
```

**Deploy the CronJob:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-merge-credentials
  namespace: vault
spec:
  schedule: "*/1 * * * *"  # Every 1 minute
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: vault-merge
          restartPolicy: OnFailure
          securityContext:
            runAsUser: 0  # Required for microdnf
          containers:
          - name: merge
            image: registry.access.redhat.com/ubi9/ubi-minimal:latest
            command:
            - /bin/bash
            - -c
            - |
              set -e
              
              # Install required tools
              microdnf install -y jq unzip wget tar gzip
              
              # Install Vault CLI
              wget -q https://releases.hashicorp.com/vault/1.15.6/vault_1.15.6_linux_amd64.zip -O /tmp/vault.zip
              cd /tmp && unzip -q vault.zip && chmod +x vault && mv vault /usr/local/bin/
              
              # Configure Vault access
              export VAULT_ADDR=http://vault.vault.svc.cluster.local:8200
              export VAULT_TOKEN=$(cat /vault-token/token)
              
              # Fetch platform credentials (auto-synced by PushSecret)
              vault kv get -format=json secret/platform-pull-secret-auto > /tmp/platform.json
              
              # Fetch customer credentials
              vault kv get -format=json secret/customer-only-registry > /tmp/customer.json
              
              # Extract dockerconfigjson from Vault response
              PLATFORM=$(cat /tmp/platform.json | jq -r '.data.data[".dockerconfigjson"]')
              CUSTOMER=$(cat /tmp/customer.json | jq -r '.data.data[".dockerconfigjson"]')
              
              # Write to temp files for jq processing
              echo "$PLATFORM" > /tmp/platform-data.json
              echo "$CUSTOMER" > /tmp/customer-data.json
              
              # Merge the auths objects
              MERGED=$(jq -s '.[0].auths + .[1].auths | {auths: .}' /tmp/platform-data.json /tmp/customer-data.json)
              
              # Write merged result back to Vault
              echo "$MERGED" > /tmp/merged.json
              vault kv put secret/final-merged .dockerconfigjson=@/tmp/merged.json
              
              echo "Merge completed successfully"
            volumeMounts:
            - name: vault-token
              mountPath: /vault-token
              readOnly: true
          volumes:
          - name: vault-token
            secret:
              secretName: vault-token
```

### Step 5: Create ExternalSecret to Pull Merged Result

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
    creationPolicy: Owner
    template:
      type: kubernetes.io/dockerconfigjson
  dataFrom:
  - extract:
      key: final-merged
```

## Verification

### Check PushSecret Status

```bash
oc get pushsecret -n openshift-config
oc describe pushsecret push-platform-credentials -n openshift-config
```

### Check CronJob Execution

```bash
# Watch CronJob status
oc get cronjob -n vault

# View recent job runs
oc get jobs -n vault --sort-by=.status.startTime

# Check logs from latest job
oc logs -n vault $(oc get pods -n vault -l job-name --sort-by=.status.startTime --no-headers | tail -1 | awk '{print $1}')
```

### Verify Vault Contents

```bash
# Check all three Vault paths
vault kv get secret/platform-pull-secret-auto  # Auto-synced by PushSecret
vault kv get secret/customer-only-registry      # Manually managed
vault kv get secret/final-merged                # CronJob output

# Count registries in merged result
vault kv get -format=json secret/final-merged | \
  jq -r '.data.data[".dockerconfigjson"]' | \
  jq '.auths | keys | length'
```

### Verify pull-secret in Cluster

```bash
# Check pull-secret content
oc get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | \
  base64 -d | jq '.auths | keys'
```

## Test Results from ARO 4.20.15

### Initial Setup

**Platform credentials (4 registries):**
```json
[
  "cloud.openshift.com",
  "quay.io",
  "registry.connect.redhat.com",
  "registry.redhat.io"
]
```

**Customer credentials (1 registry):**
```json
[
  "mycustomer.azurecr.io"
]
```

**Merged result (5 registries):**
```json
[
  "cloud.openshift.com",
  "mycustomer.azurecr.io",
  "quay.io",
  "registry.connect.redhat.com",
  "registry.redhat.io"
]
```

### Customer Credential Rotation Test

**Added second customer registry:**
```bash
vault kv put secret/customer-only-registry .dockerconfigjson='{
  "auths": {
    "mycustomer.azurecr.io": {"auth": "Y3VzdDpwYXNzMQ=="},
    "anothercustomer.azurecr.io": {"auth": "Y3VzdDpwYXNzMg=="}
  }
}'
```

**Result after 58 seconds (6 registries):**
```json
[
  "anothercustomer.azurecr.io",
  "cloud.openshift.com",
  "mycustomer.azurecr.io",
  "quay.io",
  "registry.connect.redhat.com",
  "registry.redhat.io"
]
```
