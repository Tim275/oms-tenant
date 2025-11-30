# Production Overlay - OMS Kubernetes

Production-ready configuration for deploying OMS to Talos Homelab cluster.

---

## 🎯 Quick Deploy

### Prerequisites

- [ ] Talos cluster running and accessible
- [ ] `kubectl` configured with production context
- [ ] Production secrets encrypted (see [SECRETS.md](./SECRETS.md))
- [ ] Images pushed to GHCR (GitHub Container Registry)
- [ ] DNS records configured
- [ ] Ingress controller installed (Traefik/Nginx)

### Deploy

```bash
# 1. Ensure you're in the right cluster context
kubectl config current-context
# Should show: talos-admin@homelab or similar

# 2. Create namespace
kubectl create namespace oms

# 3. Apply secrets (encrypted with SOPS)
kubectl apply -f <(sops --decrypt config/secrets.enc.yaml)

# Or with SealedSecrets
kubectl apply -f config/sealed-secrets.yaml

# 4. Deploy entire stack
kubectl apply -k overlays/production

# 5. Verify deployment
kubectl get pods -n oms
kubectl get ingress -n oms

# 6. Check service health
kubectl get deployments -n oms
kubectl logs -n oms -l app=gateway --tail=50
```

---

## 📦 What's Deployed

### Backend Services
- **Gateway** (3 replicas) - HTTP API Gateway
- **Orders** (1 replica) - Order orchestration
- **Payments** (1 replica) - Stripe integration
- **Stock** (1 replica) - Inventory management
- **Kitchen** (1 replica) - Kitchen display backend

### Frontend
- **Customer App** (1 replica) - Customer-facing UI
- **Kitchen Display** (1 replica) - Kitchen UI

### Configuration
- **ConfigMap**: Production environment variables
- **Secrets**: Encrypted Stripe, DB, RabbitMQ, Redis credentials
- **ServiceAccounts**: RBAC for each service

---

## 🔧 Configuration

### Images

All images use versioned tags from GitHub Container Registry:

```yaml
images:
- name: oms-gateway
  newName: ghcr.io/tim275/oms-gateway
  newTag: v1.0.0
# ... (see kustomization.yaml)
```

### Replicas

Production replica counts:
- **Gateway**: 3 (high availability)
- **All other services**: 1 (can scale with HPA)

### Environment Variables

Production uses **external managed services**:

- **PostgreSQL**: RDS, CloudSQL, or managed instance
  - `postgresql.database.svc.cluster.local:5432`
  - SSL required (`sslmode=require`)

- **MongoDB**: Atlas, DocumentDB
  - `mongodb+srv://cluster.mongodb.net`
  - Replica set with TLS

- **RabbitMQ**: CloudAMQP, AWS MQ
  - `amqps://rabbitmq.messaging.svc.cluster.local:5671`
  - TLS required

- **Redis**: ElastiCache, Redis Cloud
  - `redis.cache.svc.cluster.local:6379`
  - Auth enabled

- **Consul**: External or dedicated namespace
  - `consul.consul.svc.cluster.local:8500`

See [config/configmap-patch.yaml](./config/configmap-patch.yaml) for details.

---

## 🔐 Secrets Management

**⚠️ CRITICAL**: Never commit unencrypted secrets!

### Quick Start

```bash
# Option 1: SOPS (Recommended for GitOps)
cd config
./encrypt-secrets.sh

# Option 2: SealedSecrets
kubeseal < secrets.yaml > sealed-secrets.yaml
```

See [SECRETS.md](./SECRETS.md) for comprehensive guide.

### Required Secrets

- **stripe-secrets**: Production Stripe keys
  - `STRIPE_SECRET_KEY` (sk_live_...)
  - `STRIPE_ENDPOINT_SECRET` (whsec_...)

- **database-secrets**: DB credentials
  - `POSTGRES_PASSWORD`
  - `MONGODB_URI`

- **rabbitmq-secrets**: Messaging credentials
  - `RABBITMQ_USER`
  - `RABBITMQ_PASSWORD`
  - `RABBITMQ_URL`

- **redis-secrets**: Cache credentials
  - `REDIS_PASSWORD`
  - `REDIS_URL`

---

## 🌐 Ingress & DNS

### DNS Records

Point these domains to your ingress controller:

```
A    oms.timourhomelab.org        → <INGRESS_IP>
A    api.oms.timourhomelab.org    → <INGRESS_IP>
A    kitchen.oms.timourhomelab.org → <INGRESS_IP>
```

### TLS Certificates

Using cert-manager with Let's Encrypt:

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ClusterIssuer
kubectl apply -f ingress/letsencrypt-prod.yaml
```

Certificates will be auto-provisioned via HTTP-01 challenge.

---

## 📊 Monitoring & Observability

### Metrics

All services expose Prometheus metrics at `/metrics`:

```bash
# Port forward to view metrics
kubectl port-forward -n oms svc/gateway 8080:8080
curl http://localhost:8080/metrics
```

### Distributed Tracing

OTEL Collector sends traces to Jaeger:

```bash
# Access Jaeger UI
kubectl port-forward -n observability svc/jaeger 16686:16686
open http://localhost:16686
```

### Health Checks

All services expose `/health` endpoint for K8s probes:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## 🚀 Scaling

### Horizontal Pod Autoscaler

Scale based on CPU/Memory:

```bash
# Create HPA for Gateway (example)
kubectl autoscale deployment gateway -n oms \
  --cpu-percent=70 \
  --min=3 \
  --max=10

# Check HPA status
kubectl get hpa -n oms
```

### Manual Scaling

```bash
# Scale specific deployment
kubectl scale deployment gateway -n oms --replicas=5

# Scale all services
kubectl scale deployment --all -n oms --replicas=2
```

---

## 🔄 Updates & Rollouts

### Update Image Tag

```bash
# Edit kustomization.yaml
vim kustomization.yaml  # Update newTag to v1.1.0

# Apply changes
kubectl apply -k overlays/production

# Watch rollout
kubectl rollout status deployment/gateway -n oms
```

### Rollback

```bash
# View rollout history
kubectl rollout history deployment/gateway -n oms

# Rollback to previous version
kubectl rollout undo deployment/gateway -n oms

# Rollback to specific revision
kubectl rollout undo deployment/gateway -n oms --to-revision=2
```

---

## 🐛 Troubleshooting

### Pods not starting

```bash
# Check pod status
kubectl get pods -n oms

# Describe pod for events
kubectl describe pod <pod-name> -n oms

# Check logs
kubectl logs -n oms <pod-name> --tail=100

# Check previous crashed container
kubectl logs -n oms <pod-name> --previous
```

### Secrets not found

```bash
# List secrets
kubectl get secrets -n oms

# Check secret content
kubectl get secret stripe-secrets -n oms -o yaml

# Verify secret keys match deployment
kubectl get deployment gateway -n oms -o yaml | grep -A 5 secretKeyRef
```

### Service discovery issues

```bash
# Check services
kubectl get svc -n oms

# Test DNS resolution from pod
kubectl run -it --rm debug --image=nicolaka/netshoot -n oms -- /bin/bash
nslookup orders.oms.svc.cluster.local
curl http://orders.oms.svc.cluster.local:9000

# Check Consul registration
kubectl port-forward -n oms svc/consul 8500:8500
curl http://localhost:8500/v1/catalog/services
```

### Ingress not working

```bash
# Check ingress
kubectl get ingress -n oms

# Describe ingress for events
kubectl describe ingress oms-ingress -n oms

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

---

## 📋 Pre-Production Checklist

Before going live:

- [ ] **Secrets**: All production secrets encrypted and applied
- [ ] **DNS**: All domains resolving to ingress
- [ ] **TLS**: Certificates issued and valid
- [ ] **Database**: Production databases created and accessible
- [ ] **RabbitMQ**: Production broker configured with TLS
- [ ] **Redis**: Production cache configured with auth
- [ ] **Monitoring**: Prometheus scraping all services
- [ ] **Tracing**: Jaeger receiving traces
- [ ] **Backup**: Database backup strategy in place
- [ ] **Alerts**: Alertmanager configured for critical errors
- [ ] **Load Testing**: Tested with expected production load
- [ ] **Disaster Recovery**: Tested restore from backups
- [ ] **Documentation**: Runbooks for common issues

---

## 🔗 External Resources

- **GitHub Container Registry**: https://github.com/tim275?tab=packages
- **Stripe Dashboard**: https://dashboard.stripe.com
- **Talos Cluster**: SSH into control plane for cluster operations

---

## 📞 Support

For issues:
1. Check [Troubleshooting](#troubleshooting) section
2. Review service logs: `kubectl logs -n oms -l app=<service> --tail=100`
3. Check events: `kubectl get events -n oms --sort-by='.lastTimestamp'`

---

**Environment**: Production (Talos Homelab)
**Namespace**: `oms`
**Maintained by**: Timour
**Last Updated**: November 18, 2025
