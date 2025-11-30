# OMS Production Deployment Guide - Talos Homelab

Complete guide for deploying OMS to your Talos homelab cluster with SealedSecrets.

---

## ✅ Pre-Deployment Checklist

### Infrastructure Ready

- [x] **Talos Cluster**: Running and accessible (`admin@homelab-k8s`)
- [x] **SealedSecrets Controller**: Running in `sealed-secrets` namespace
- [x] **Public Key**: Fetched and saved (`sealed-secrets-pub.crt`)
- [ ] **Namespace**: Create `oms` namespace
- [ ] **Ingress Controller**: Traefik/Nginx installed
- [ ] **Cert-Manager**: For TLS certificates (optional but recommended)

### Secrets Prepared

- [ ] **Stripe Production Keys**: From https://dashboard.stripe.com/apikeys
  - `STRIPE_SECRET_KEY` (starts with `sk_live_`)
  - `STRIPE_ENDPOINT_SECRET` (webhook secret)

- [ ] **Database Credentials**: Strong passwords generated
  - PostgreSQL: `openssl rand -base64 32`
  - MongoDB Atlas URI (if using Atlas)

- [ ] **RabbitMQ Credentials**: Production user + password
  - User: `oms-production` (NOT `guest`!)
  - Password: `openssl rand -base64 32`
  - URL: CloudAMQP or self-hosted with TLS

- [ ] **Redis Credentials**: Password for production
  - ElastiCache, Redis Cloud, or self-hosted

### Images Built & Pushed

- [ ] All service images built and pushed to GHCR
  ```bash
  # Example for Gateway
  docker build -t ghcr.io/tim275/oms-gateway:v1.0.0 -f gateway/Dockerfile .
  docker push ghcr.io/tim275/oms-gateway:v1.0.0
  ```

---

## 🚀 Deployment Steps

### Step 1: Prepare Secrets

```bash
# Navigate to config directory
cd /Users/timour/Desktop/Golang/oms-k8s/overlays/production/config

# Edit secrets template
cp secrets.yaml.template secrets.yaml
vim secrets.yaml

# Fill in ALL CHANGEME placeholders with actual production values:
# - STRIPE_SECRET_KEY: sk_live_xxxxxxxxxxxxx
# - STRIPE_ENDPOINT_SECRET: whsec_xxxxxxxxxxxxx
# - POSTGRES_PASSWORD: (generated strong password)
# - MONGODB_URI: mongodb+srv://user:pass@cluster.mongodb.net/orders
# - RABBITMQ_USER: oms-production
# - RABBITMQ_PASSWORD: (generated strong password)
# - RABBITMQ_URL: amqps://user:pass@host:5671/
# - REDIS_PASSWORD: (generated strong password)
# - REDIS_URL: redis://:password@host:6379/0
```

### Step 2: Seal Secrets

```bash
# Run automated sealing script
./seal-secrets.sh

# Expected output:
# ✅ Secrets sealed successfully!
# 📦 Sealed Secrets:
#   - stripe-secrets
#   - database-secrets
#   - rabbitmq-secrets
#   - redis-secrets
```

### Step 3: Delete Unencrypted Secrets

```bash
# IMPORTANT: Remove unencrypted secrets file
rm secrets.yaml

# Verify sealed secrets exist
ls -lh sealed-secrets.yaml

# Should be ~9KB with encrypted data
```

### Step 4: Review Configuration

```bash
# Check ConfigMap for production environment variables
cat ../configmap-patch.yaml

# Verify:
# - POSTGRES_HOST points to your managed PostgreSQL
# - MONGODB_URI points to MongoDB Atlas/managed instance
# - RABBITMQ_URL uses amqps:// (TLS)
# - All service endpoints correct
```

### Step 5: Switch to Production Context

```bash
# List available contexts
kubectl config get-contexts

# Switch to homelab cluster
kubectl config use-context admin@homelab-k8s

# Verify
kubectl config current-context
# Should show: admin@homelab-k8s

# Test connection
kubectl get nodes
```

### Step 6: Create Namespace

```bash
# Create OMS namespace
kubectl create namespace oms

# Verify
kubectl get namespace oms
```

### Step 7: Apply Sealed Secrets

```bash
# Apply sealed secrets to cluster
kubectl apply -f sealed-secrets.yaml

# Verify secrets were created (SealedSecrets controller decrypts them)
kubectl get secrets -n oms

# Expected output:
# NAME                TYPE     DATA   AGE
# stripe-secrets      Opaque   2      10s
# database-secrets    Opaque   3      10s
# rabbitmq-secrets    Opaque   3      10s
# redis-secrets       Opaque   2      10s

# Check one secret was decrypted correctly
kubectl get secret stripe-secrets -n oms -o yaml
# Should show STRIPE_SECRET_KEY and STRIPE_ENDPOINT_SECRET in data section
```

### Step 8: Deploy OMS Application

```bash
# Navigate to oms-k8s directory
cd /Users/timour/Desktop/Golang/oms-k8s

# Preview what will be deployed
kubectl kustomize overlays/production | less

# Apply everything
kubectl apply -k overlays/production

# Expected output:
# serviceaccount/gateway created
# serviceaccount/orders created
# ... (more serviceaccounts)
# configmap/oms-config created
# service/gateway created
# service/orders created
# ... (more services)
# deployment.apps/gateway created
# deployment.apps/orders created
# ... (more deployments)
```

### Step 9: Monitor Deployment

```bash
# Watch pods starting
kubectl get pods -n oms -w

# Wait for all pods to be Running (Ctrl+C to stop watching)
# Expected:
# NAME                              READY   STATUS    RESTARTS   AGE
# gateway-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
# gateway-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
# gateway-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
# orders-xxxxxxxxxx-xxxxx           1/1     Running   0          2m
# payments-xxxxxxxxxx-xxxxx         1/1     Running   0          2m
# stock-xxxxxxxxxx-xxxxx            1/1     Running   0          2m
# kitchen-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
# customer-app-xxxxxxxxxx-xxxxx    1/1     Running   0          2m
# kitchen-display-xxxxxxxxxx-xxxxx 1/1     Running   0          2m

# Check deployments
kubectl get deployments -n oms

# Check services
kubectl get svc -n oms
```

### Step 10: Verify Services

```bash
# Check Gateway health
kubectl port-forward -n oms svc/gateway 8080:8080 &
curl http://localhost:8080/health
# Expected: {"status":"healthy"}

# Check Gateway metrics
curl http://localhost:8080/metrics | grep -E "http_requests|grpc_client"

# Check Orders health
kubectl port-forward -n oms svc/orders 9001:9001 &
curl http://localhost:9001/health
# Expected: {"status":"healthy"}

# Check logs
kubectl logs -n oms -l app=gateway --tail=50
kubectl logs -n oms -l app=orders --tail=50
kubectl logs -n oms -l app=payments --tail=50
```

### Step 11: Configure Ingress (Optional)

```bash
# If using Traefik/Nginx, create Ingress
cat > ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: oms-ingress
  namespace: oms
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.middlewares: oms-cors@kubernetescrd
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - oms.timourhomelab.org
    - api.oms.timourhomelab.org
    - kitchen.oms.timourhomelab.org
    secretName: oms-tls
  rules:
  - host: oms.timourhomelab.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: customer-app
            port:
              number: 80
  - host: api.oms.timourhomelab.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gateway
            port:
              number: 8080
  - host: kitchen.oms.timourhomelab.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kitchen-display
            port:
              number: 80
EOF

kubectl apply -f ingress.yaml

# Verify ingress created
kubectl get ingress -n oms
```

### Step 12: Test End-to-End Flow

```bash
# Test from outside cluster (if ingress configured)
curl https://api.oms.timourhomelab.org/api/menu

# Or port-forward for testing
kubectl port-forward -n oms svc/gateway 8080:8080

# Test menu endpoint
curl http://localhost:8080/api/menu | jq

# Test create order
curl -X POST http://localhost:8080/api/orders/create \
  -H "Content-Type: application/json" \
  -d '{
    "customer_id": "test-customer",
    "items": [
      {"item_id": "burger", "quantity": 2},
      {"item_id": "fries", "quantity": 1}
    ]
  }' | jq

# Expected response:
# {
#   "order_id": "...",
#   "payment_link": "https://checkout.stripe.com/..."
# }
```

---

## 🔧 Post-Deployment Configuration

### Stripe Webhook

1. Go to https://dashboard.stripe.com/webhooks
2. Click "Add endpoint"
3. Endpoint URL: `https://api.oms.timourhomelab.org/webhook/stripe`
4. Events to select: `checkout.session.completed`
5. Click "Add endpoint"
6. Copy webhook secret (starts with `whsec_`)
7. Update sealed secrets with new webhook secret
8. Re-seal and apply

### Monitoring

```bash
# Check Prometheus metrics
kubectl port-forward -n oms svc/gateway 8080:8080
curl http://localhost:8080/metrics

# Access Jaeger (if deployed)
kubectl port-forward -n observability svc/jaeger 16686:16686
open http://localhost:16686
```

---

## 🐛 Troubleshooting

### Pods CrashLooping

```bash
# Check pod logs
kubectl logs -n oms <pod-name> --tail=100

# Check previous crashed container
kubectl logs -n oms <pod-name> --previous

# Describe pod for events
kubectl describe pod -n oms <pod-name>

# Common issues:
# - Secrets not found → Check: kubectl get secrets -n oms
# - Database connection failed → Check POSTGRES_URI/MONGODB_URI
# - RabbitMQ connection failed → Check RABBITMQ_URL (use amqps://)
```

### Secrets Not Decrypted

```bash
# Check SealedSecrets controller running
kubectl get pods -n sealed-secrets

# Check SealedSecret resource
kubectl get sealedsecrets -n oms

# Check events
kubectl get events -n oms --sort-by='.lastTimestamp' | grep -i seal

# If sealed-secrets controller is missing, install:
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml
```

### Service Discovery Issues

```bash
# Test DNS resolution from pod
kubectl run -it --rm debug --image=nicolaka/netshoot -n oms -- /bin/bash
nslookup orders.oms.svc.cluster.local
curl http://orders.oms.svc.cluster.local:9000

# Check services
kubectl get svc -n oms

# Check endpoints
kubectl get endpoints -n oms
```

---

## 📊 Production Readiness Checklist

- [ ] All pods Running (0 CrashLoopBackOff)
- [ ] Health checks passing (`/health` returns 200)
- [ ] Secrets applied and decrypted
- [ ] Database connections working (check logs)
- [ ] RabbitMQ connections established (check logs)
- [ ] Ingress accessible from internet
- [ ] TLS certificates issued (if using cert-manager)
- [ ] Stripe webhook configured and receiving events
- [ ] End-to-end order flow tested
- [ ] Monitoring/metrics accessible
- [ ] Logs aggregated (if using Loki/ELK)
- [ ] Backup strategy in place for databases
- [ ] Disaster recovery plan documented

---

## 🔄 Updates & Rollbacks

### Update Service

```bash
# Edit kustomization.yaml
vim overlays/production/kustomization.yaml

# Change image tag:
# - name: oms-gateway
#   newTag: v1.1.0  # Changed from v1.0.0

# Apply changes
kubectl apply -k overlays/production

# Watch rollout
kubectl rollout status deployment/gateway -n oms

# Check new pods
kubectl get pods -n oms -l app=gateway
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

## 📚 Additional Resources

- **SECRETS.md**: Comprehensive secrets management guide
- **README.md**: Production overlay overview
- **Talos Docs**: https://www.talos.dev/
- **SealedSecrets**: https://github.com/bitnami-labs/sealed-secrets
- **Kustomize**: https://kustomize.io/

---

**Deployment Date**: _____________
**Deployed By**: Timour
**Cluster**: homelab-k8s (Talos)
**Version**: v1.0.0
