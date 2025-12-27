# Testing Manifests for K8SPS-632 (AWS Session Token Support)

These manifests were used to manually test the S3_SESSION_TOKEN fix for backup, restore, and PITR operations.

## Prerequisites

1. EKS cluster with PXC operator deployed
2. External Secrets Operator installed (for session token test)
3. Token Vending Machine service available at `http://tvm-backup-creds.token-vending-machine/credentials`

## Test Scenarios

### 1. Session Token Test (files 01-03)

Tests backup/restore/PITR with AWS credentials that include `AWS_SESSION_TOKEN`.

```bash
# Create namespace
kubectl create namespace portal

# Apply prerequisites (ServiceAccount, Webhook, ExternalSecret)
kubectl apply -f 01-prerequisites.yaml

# Wait for secret to be created
kubectl get secret portal-blueprint-backup-creds -n portal

# Deploy PXC cluster with session token credentials
kubectl apply -f 02-pxc-clean-session-token.yaml

# Insert test data
kubectl exec pxc-clean-pxc-0 -n portal -- mysql -uroot -p<password> -e \
  "CREATE DATABASE testdb; USE testdb; CREATE TABLE events (id INT AUTO_INCREMENT PRIMARY KEY, msg VARCHAR(100)); INSERT INTO events (msg) VALUES ('pre-backup');"

# Create backup
kubectl apply -f 03-backup-restore-session-token.yaml  # Apply backup CR only first

# Insert post-backup data
kubectl exec pxc-clean-pxc-0 -n portal -- mysql -uroot -p<password> -e \
  "USE testdb; INSERT INTO events (msg) VALUES ('post-backup');"

# Wait for binlog upload, then apply DR cluster and restore
kubectl apply -f 03-backup-restore-session-token.yaml  # Apply full file

# Verify restored data includes post-backup rows
kubectl exec pxc-dr-pxc-0 -n portal -- mysql -uroot -p<password> -e "USE testdb; SELECT * FROM events;"
```

### 2. Static Credentials Test (files 04-05)

Tests backward compatibility with static AWS credentials (no session token).

```bash
# Copy static credentials from monitoring namespace
kubectl get secret backup-creds -n monitoring -o json | \
  jq 'del(.metadata.namespace, .metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp, .metadata.managedFields) | .metadata.name = "static-backup-creds"' | \
  kubectl apply -n portal -f -

# Deploy PXC cluster with static credentials
kubectl apply -f 04-pxc-static-credentials.yaml

# Insert test data, create backup, insert more data (same flow as above)
# ...

# Create DR cluster and restore
kubectl apply -f 05-backup-restore-static-credentials.yaml

# Verify restored data
kubectl exec pxc-static-dr-pxc-0 -n portal -- mysql -uroot -p<password> -e "USE testdb; SELECT * FROM events;"
```

## Environment-Specific Values

Update these values for your environment:

- `ghcr.io/gmautner/pxc-operator:K8SPS-632-test` - Custom operator image
- `lwsa-idp-testbed-dev-backup-eks-06-ttzun9n9` - S3 bucket name
- `us-east-2` - AWS region
- Backup destination paths (timestamps will differ)

## Cleanup

```bash
kubectl delete pxc --all -n portal
kubectl delete pxc-backup --all -n portal
kubectl delete pxc-restore --all -n portal
kubectl delete namespace portal
```

