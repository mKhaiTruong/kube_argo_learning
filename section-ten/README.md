## ArgoCD
GitOps CD tool — tự động sync từ Git repo xuống cluster. Git là source of truth duy nhất, không kubectl apply thủ công trong production.

```yaml
syncPolicy:
  automated:
    prune: true      # xóa resource không còn trong Git
    selfHeal: true   # tự fix nếu ai sửa thẳng trên cluster
  syncOptions:
    - CreateNamespace=true
```

#### Kustomization
Giải quyết vấn đề deploy nhiều môi trường mà không copy paste yaml. Base chứa yaml dùng chung, overlay chỉ override những gì khác:

```
base/
├── deployment.yaml
└── kustomization.yaml

overlays/
├── staging/
│   └── kustomization.yaml  (replicas: 1)
└── production/
    └── kustomization.yaml  (replicas: 5)
```
ArgoCD trỏ vào overlays/staging hay overlays/production tùy môi trường.

---

## Helm
Package manager cho k8s — gộp tất cả yaml thành 1 chart, dùng Go template syntax {{ }} để inject values:

```
chart/
├── Chart.yaml       # metadata
├── values.yaml      # giá trị mặc định
└── templates/       # yaml + {{ }} placeholders
```

Deploy nhiều môi trường chỉ cần đổi values file:
```
bashhelm install app ./chart -f values-prod.yaml
helm install app ./chart -f values-staging.yaml
```

*Kustomize vs Helm*: Kustomize đơn giản hơn, Helm mạnh hơn nhưng phức tạp hơn. Nhiều team dùng cả 2 tùy use case.

---