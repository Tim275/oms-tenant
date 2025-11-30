# OMS Kubernetes Manifests

Production-ready Kubernetes manifests for Order Management System.

## Structure

```
├── base/              # Base manifests (common across environments)
├── overlays/          # Environment-specific configurations
│   ├── local-k3d/    # Local development
│   ├── dev/          # Dev cluster
│   └── production/   # Production cluster
└── infrastructure/   # Infrastructure components
```

## Deployment

### Local k3d
```bash
kubectl apply -k overlays/local-k3d
```

### Dev
```bash
kubectl apply -k overlays/dev
```

### Production
```bash
kubectl apply -k overlays/production
```

## Images

All images: `ghcr.io/tim275/oms-*:v1.0.0`

## Security

- Non-root containers
- Read-only root filesystem
- Capabilities dropped
- Resource limits enforced

## TODO

- [ ] Network Policies (Zero-Trust zwischen Services)
- [ ] Pod Anti-Affinity in Deployments
- [ ] Istio Rate Limiting (EnvoyFilter)
- [ ] Istio Circuit Breaker (DestinationRule)
