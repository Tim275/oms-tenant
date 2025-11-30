# Production Secrets Management

⚠️ **CRITICAL**: Never commit unencrypted secrets to git! ⚠️

This guide covers production-ready secrets management for the OMS application.

---

## 📋 Table of Contents

1. [Quick Start](#quick-start)
2. [SOPS Encryption (Recommended)](#sops-encryption-recommended)
3. [SealedSecrets Alternative](#sealedsecrets-alternative)
4. [External Secrets Operator](#external-secrets-operator)
5. [Secrets Checklist](#secrets-checklist)
6. [Troubleshooting](#troubleshooting)

---

## 🚀 Quick Start

### Option 1: SealedSecrets (✅ Already Configured!)

**Your cluster already has SealedSecrets controller running!**

```bash
# 1. Navigate to config directory
cd overlays/production/config

# 2. Edit secrets with actual production values
cp secrets.yaml.template secrets.yaml
vim secrets.yaml  # Replace all CHANGEME placeholders

# 3. Run the automated sealing script
./seal-secrets.sh

# Output:
# ✅ Secrets sealed successfully!
# 📦 Sealed Secrets: stripe-secrets, database-secrets, rabbitmq-secrets, redis-secrets

# 4. Delete unencrypted file
rm secrets.yaml

# 5. Commit sealed version (SAFE to commit!)
git add sealed-secrets.yaml
git commit -m "Add production sealed secrets"

# 6. Apply to cluster
kubectl apply -f sealed-secrets.yaml -n oms
```

**Why SealedSecrets?**
✅ Already running in your cluster (`sealed-secrets` namespace)
✅ Public key already fetched (`sealed-secrets-pub.crt`)
✅ One-way encryption - only cluster can decrypt
✅ Safe to commit to git
✅ Automated script ready to use

### Option 2: SOPS (Alternative for GitOps)

```bash
# 1. Install SOPS and age
brew install sops age

# 2. Generate age key pair
age-keygen -o ~/.config/sops/age/keys.txt
# Output: Public key: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 3. Update .sops.yaml with your public key
vim .sops.yaml  # Replace age1xxx... with your public key

# 4. Copy secrets template
cd overlays/production/config
cp secrets.yaml.template secrets.yaml

# 5. Fill in actual production values
vim secrets.yaml  # Replace all CHANGEME placeholders

# 6. Encrypt secrets
sops --encrypt secrets.yaml > secrets.enc.yaml

# 7. Delete unencrypted file
rm secrets.yaml

# 8. Commit encrypted version
git add secrets.enc.yaml .sops.yaml
git commit -m "Add encrypted production secrets"
git push

# 9. Deploy to cluster
kubectl apply -f <(sops --decrypt secrets.enc.yaml)
```

### Manual SealedSecrets (Without Script)

```bash
# 1. Install Sealed Secrets controller in cluster
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# 2. Install kubeseal CLI
brew install kubeseal

# 3. Prepare secrets
cd overlays/production/config
cp secrets.yaml.template secrets.yaml
vim secrets.yaml  # Fill in actual values

# 4. Seal the secrets
kubeseal --format=yaml < secrets.yaml > sealed-secrets.yaml

# 5. Delete unencrypted file
rm secrets.yaml

# 6. Commit sealed version
git add sealed-secrets.yaml
git commit -m "Add sealed production secrets"

# 7. Deploy
kubectl apply -f sealed-secrets.yaml
```

---

## 🔐 SOPS Encryption (Recommended)

### Why SOPS?

✅ **GitOps-friendly**: Encrypted secrets can be committed to git
✅ **Diff-friendly**: Changes are visible in git history
✅ **Flexible**: Supports age, PGP, AWS KMS, GCP KMS, Azure Key Vault
✅ **Partial encryption**: Only encrypts sensitive fields, metadata visible
✅ **CI/CD integration**: Works with ArgoCD, FluxCD, etc.

### Setup

#### 1. Install SOPS and age

```bash
# macOS
brew install sops age

# Linux
curl -LO https://github.com/getsops/sops/releases/latest/download/sops-v3.8.1.linux.amd64
sudo mv sops-v3.8.1.linux.amd64 /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops

# age
curl -LO https://github.com/FiloSottile/age/releases/latest/download/age-v1.1.1-linux-amd64.tar.gz
tar xzf age-v1.1.1-linux-amd64.tar.gz
sudo mv age/age /usr/local/bin/
```

#### 2. Generate age key

```bash
# Create directory
mkdir -p ~/.config/sops/age

# Generate key pair
age-keygen -o ~/.config/sops/age/keys.txt

# Output:
# Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
# (save this!)

# Extract public key
age-keygen -y ~/.config/sops/age/keys.txt
```

#### 3. Configure SOPS

Edit `.sops.yaml` in the repository root:

```yaml
creation_rules:
  - path_regex: overlays/production/.*secrets.*\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p  # Your public key
```

#### 4. Encrypt secrets

```bash
# Encrypt file
sops --encrypt config/secrets.yaml > config/secrets.enc.yaml

# Decrypt file (for applying)
sops --decrypt config/secrets.enc.yaml

# Edit encrypted file
sops config/secrets.enc.yaml

# Apply to cluster
kubectl apply -f <(sops --decrypt config/secrets.enc.yaml)
```

#### 5. Share with team

**Share private key securely** (1Password, Vault, etc.):

```bash
# Export private key
cat ~/.config/sops/age/keys.txt

# Team member imports:
mkdir -p ~/.config/sops/age
vim ~/.config/sops/age/keys.txt  # Paste private key
chmod 600 ~/.config/sops/age/keys.txt
```

### SOPS with ArgoCD

```yaml
# Install SOPS plugin for ArgoCD
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  kustomize.buildOptions: --enable-alpha-plugins
```

```bash
# Add age key to ArgoCD repo server
kubectl create secret generic sops-age \
  --from-file=keys.txt=$HOME/.config/sops/age/keys.txt \
  -n argocd

# Update ArgoCD repo server deployment
kubectl patch deployment argocd-repo-server -n argocd --patch '
spec:
  template:
    spec:
      containers:
      - name: argocd-repo-server
        env:
        - name: SOPS_AGE_KEY_FILE
          value: /sops-age/keys.txt
        volumeMounts:
        - name: sops-age
          mountPath: /sops-age
      volumes:
      - name: sops-age
        secret:
          secretName: sops-age
'
```

---

## 🔒 SealedSecrets Alternative

### Why SealedSecrets?

✅ **Kubernetes-native**: No external dependencies
✅ **Simple**: Encrypt/decrypt with cluster keypair
✅ **One-way encryption**: Only cluster can decrypt
✅ **No shared keys**: Each cluster has unique keypair

### Setup

#### 1. Install Sealed Secrets controller

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Verify
kubectl get pods -n kube-system | grep sealed-secrets
```

#### 2. Install kubeseal CLI

```bash
# macOS
brew install kubeseal

# Linux
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar xfz kubeseal-0.24.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

#### 3. Seal secrets

```bash
# Prepare secret
cd overlays/production/config
cp secrets.yaml.template secrets.yaml
vim secrets.yaml  # Fill in actual values

# Seal it
kubeseal --format=yaml < secrets.yaml > sealed-secrets.yaml

# Delete original
rm secrets.yaml

# Commit sealed version
git add sealed-secrets.yaml
git commit -m "Add sealed secrets"
```

#### 4. Deploy

```bash
# Apply sealed secret
kubectl apply -f sealed-secrets.yaml

# Controller automatically creates Secret
kubectl get secrets -n oms | grep stripe
```

### Backup controller key

⚠️ **Important**: Backup the controller keypair! Without it, you can't decrypt!

```bash
# Backup encryption key
kubectl get secret -n kube-system sealed-secrets-key -o yaml > sealed-secrets-key.backup.yaml

# Store in secure location (1Password, Vault, encrypted USB)
```

---

## ☁️ External Secrets Operator

### Why External Secrets?

✅ **Cloud-native**: Integrates with AWS/GCP/Azure secret managers
✅ **No git commits**: Secrets stay in cloud provider
✅ **Auto-sync**: Automatically refreshes from source
✅ **Centralized**: One source of truth for all environments

### Setup (AWS Secrets Manager Example)

#### 1. Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace
```

#### 2. Create secrets in AWS Secrets Manager

```bash
# Stripe secrets
aws secretsmanager create-secret \
  --name production/oms/stripe \
  --secret-string '{
    "secret_key": "sk_live_xxxxxxxxxxxxx",
    "endpoint_secret": "whsec_xxxxxxxxxxxxxx"
  }'

# Database secrets
aws secretsmanager create-secret \
  --name production/oms/database \
  --secret-string '{
    "postgres_password": "xxxxxxxx",
    "mongodb_uri": "mongodb+srv://..."
  }'
```

#### 3. Create SecretStore

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: oms
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

#### 4. Create ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: stripe-secrets-external
  namespace: oms
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: stripe-secrets
    creationPolicy: Owner
  data:
  - secretKey: STRIPE_SECRET_KEY
    remoteRef:
      key: production/oms/stripe
      property: secret_key
  - secretKey: STRIPE_ENDPOINT_SECRET
    remoteRef:
      key: production/oms/stripe
      property: endpoint_secret
```

---

## ✅ Secrets Checklist

Before deploying to production, ensure:

### Stripe Secrets

- [ ] **STRIPE_SECRET_KEY**: Production key (starts with `sk_live_`)
  - Get from: https://dashboard.stripe.com/apikeys
- [ ] **STRIPE_ENDPOINT_SECRET**: Webhook secret (starts with `whsec_`)
  - Create webhook: https://dashboard.stripe.com/webhooks
  - Endpoint URL: `https://api.oms.yourdomain.com/webhook/stripe`
  - Event: `checkout.session.completed`

### Database Secrets

- [ ] **POSTGRES_PASSWORD**: Strong password (32+ chars)
  - Generate: `openssl rand -base64 32`
- [ ] **MONGODB_URI**: Complete connection string
  - Example: `mongodb+srv://user:pass@cluster.mongodb.net/orders`
  - MongoDB Atlas: https://cloud.mongodb.com

### RabbitMQ Secrets

- [ ] **RABBITMQ_USER**: Production user (NOT `guest`!)
- [ ] **RABBITMQ_PASSWORD**: Strong password (32+ chars)
- [ ] **RABBITMQ_URL**: Complete AMQPS URL
  - Use TLS in production: `amqps://` (not `amqp://`)

### Redis Secrets

- [ ] **REDIS_PASSWORD**: Strong password
- [ ] **REDIS_URL**: Complete connection string with auth

### Encryption

- [ ] Secrets encrypted with SOPS, SealedSecrets, or stored in cloud provider
- [ ] `.gitignore` blocks unencrypted `secrets.yaml`
- [ ] Encryption keys backed up securely
- [ ] Team members have access to decrypt

---

## 🐛 Troubleshooting

### SOPS: "Failed to get the data key"

**Problem**: Can't decrypt because private key missing.

**Solution**:
```bash
# Check age key exists
ls ~/.config/sops/age/keys.txt

# Check SOPS can find it
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
sops --decrypt secrets.enc.yaml
```

### SealedSecrets: "No private key found"

**Problem**: Sealed Secrets controller not running or lost keypair.

**Solution**:
```bash
# Check controller running
kubectl get pods -n kube-system | grep sealed-secrets

# Check keypair exists
kubectl get secret -n kube-system sealed-secrets-key

# Restore from backup if lost
kubectl apply -f sealed-secrets-key.backup.yaml
```

### "Secret not found" in pods

**Problem**: Secret not created or wrong namespace.

**Solution**:
```bash
# Check secret exists
kubectl get secrets -n oms | grep stripe

# Check secret content
kubectl get secret stripe-secrets -n oms -o yaml

# Check pod reference
kubectl get deployment payments -n oms -o yaml | grep -A 5 secretKeyRef
```

### Can't apply encrypted secrets to cluster

**SOPS**:
```bash
# Decrypt on-the-fly
kubectl apply -f <(sops --decrypt secrets.enc.yaml)

# Or decrypt to file first
sops --decrypt secrets.enc.yaml > /tmp/secrets.yaml
kubectl apply -f /tmp/secrets.yaml
rm /tmp/secrets.yaml
```

**SealedSecrets**:
```bash
# Just apply sealed version directly
kubectl apply -f sealed-secrets.yaml
```

---

## 📚 Resources

- **SOPS**: https://github.com/getsops/sops
- **age encryption**: https://github.com/FiloSottile/age
- **SealedSecrets**: https://github.com/bitnami-labs/sealed-secrets
- **External Secrets**: https://external-secrets.io
- **ArgoCD SOPS**: https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/

---

## 🔑 Security Best Practices

1. **Never commit unencrypted secrets to git**
2. **Use strong passwords**: 32+ characters, generated randomly
3. **Rotate secrets regularly**: Every 90 days minimum
4. **Use TLS for databases/messaging**: `postgresql://...?sslmode=require`, `amqps://`
5. **Principle of least privilege**: Each service gets only the secrets it needs
6. **Audit access**: Log who decrypts/accesses secrets
7. **Backup encryption keys**: Store in multiple secure locations
8. **Use production keys only in production**: Never use prod keys in dev/staging!

---

**Last Updated**: November 18, 2025
**Status**: Production Ready
**Maintained by**: Timour
