# Platform Admin Resources

Diese Dateien gehören NICHT zum Tenant, sondern werden vom Platform Admin (dein Homelab) deployed.

## Verwendung

In deinem Homelab Repository:

```
homelab/
├── infrastructure/
│   └── tenants/
│       └── oms/
│           ├── namespace.yaml
│           ├── resourcequota.yaml    # Aus diesem Ordner
│           └── limitrange.yaml       # Aus diesem Ordner
```

## Deployment durch Platform Admin

```bash
# Platform Admin deployed zuerst die Quotas
kubectl apply -f platform-admin/

# Dann kann der Tenant deployed werden (via ArgoCD)
kubectl apply -f argocd-application-prod.yaml
```

## Was macht das?

- **ResourceQuota**: Limitiert den gesamten oms Namespace
- **LimitRange**: Setzt Defaults für Pods/Container

## Wer deployed was?

| Was | Wer deployed? |
|-----|---------------|
| Namespace | Platform Admin |
| ResourceQuota | Platform Admin |
| LimitRange | Platform Admin |
| OMS Apps | Tenant (via ArgoCD) |
