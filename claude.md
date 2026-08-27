# Nhiệm vụ: Thêm HPA (autoscaling) vào repo GitOps

Đây là hướng dẫn thay đổi cho repo `gitops-demo` (Argo CD + Kustomize, 3 môi trường
dev/staging/prod). Hãy thực hiện chính xác các thay đổi bên dưới. Không đổi thêm gì ngoài
danh sách này.

## Bối cảnh / lý do (đọc để hiểu, đừng làm sai)

- Repo dùng cấu trúc `app/base` (dùng chung) + `app/overlays/{dev,staging,prod}` (khác biệt
  theo môi trường). Mỗi môi trường có một Argo CD Application ở `argocd/<env>.yaml`.
- Ta thêm HorizontalPodAutoscaler (HPA) cho `backend` và `frontend`, ngưỡng khác nhau theo
  môi trường.
- **Vấn đề then chốt phải xử lý:** HPA và Argo CD sẽ tranh nhau field `spec.replicas` của
  Deployment. Nếu không xử lý, HPA scale lên → Argo `selfHeal` kéo về giá trị trong Git →
  giằng co vô tận. Cách giải quyết (BẮT BUỘC làm cả hai):
  1. Bỏ hẳn `replicas` khỏi Deployment trong `app/base/` để HPA toàn quyền quản.
  2. Thêm `ignoreDifferences` cho `/spec/replicas` trong mỗi Application để Argo CD bỏ qua.
- **Không** đụng tới sealed secret, ingress, service, namespace.

## Tổng quan thay đổi

| Hành động | File |
|-----------|------|
| ➕ THÊM | `app/overlays/dev/hpa.yaml` |
| ➕ THÊM | `app/overlays/staging/hpa.yaml` |
| ➕ THÊM | `app/overlays/prod/hpa.yaml` |
| ➕ THÊM | `app/overlays/prod/resources-patch.yaml` |
| ✏️ SỬA | `app/base/backend.yaml` (bỏ dòng `replicas`) |
| ✏️ SỬA | `app/base/frontend.yaml` (bỏ dòng `replicas`) |
| ✏️ SỬA | `app/overlays/dev/kustomization.yaml` |
| ✏️ SỬA | `app/overlays/staging/kustomization.yaml` |
| ✏️ SỬA | `app/overlays/prod/kustomization.yaml` |
| ✏️ SỬA | `argocd/dev.yaml` |
| ✏️ SỬA | `argocd/staging.yaml` |
| ✏️ SỬA | `argocd/prod.yaml` |
| ❌ XÓA | `app/overlays/dev/replicas-patch.yaml` |
| ❌ XÓA | `app/overlays/staging/replicas-patch.yaml` |
| ❌ XÓA | `app/overlays/prod/replicas-patch.yaml` |

---

## 1. ➕ THÊM — `app/overlays/dev/hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 1
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 1
  maxReplicas: 2
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
```

## 2. ➕ THÊM — `app/overlays/staging/hpa.yaml`

Giống dev nhưng đổi ngưỡng: backend `minReplicas: 2 / maxReplicas: 5`,
frontend `minReplicas: 2 / maxReplicas: 4`, cả hai `averageUtilization: 75`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 75
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 75
```

## 3. ➕ THÊM — `app/overlays/prod/hpa.yaml`

Ngưỡng prod: backend `minReplicas: 3 / maxReplicas: 10`,
frontend `minReplicas: 3 / maxReplicas: 8`, cả hai `averageUtilization: 70`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 3
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## 4. ➕ THÊM — `app/overlays/prod/resources-patch.yaml`

Đây là nội dung tách ra từ `replicas-patch.yaml` cũ của prod (bỏ phần `replicas`, giữ phần
`resources`). Backend prod cần request/limit cao hơn base.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    spec:
      containers:
        - name: backend
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { cpu: "1",    memory: "1Gi" }
```

---

## 5. ✏️ SỬA — `app/base/backend.yaml` và `app/base/frontend.yaml`

Trong phần Deployment, XÓA đúng dòng `replicas`. HPA sẽ quản field này. Ví dụ:

```diff
 spec:
-  replicas: 1
   selector:
     matchLabels:
       app: backend
```

Làm cho CẢ HAI file `backend.yaml` và `frontend.yaml`.
LƯU Ý: giữ nguyên `resources.requests.cpu` trong container — HPA theo CPU% BẮT BUỘC cần
`requests.cpu` để tính phần trăm. Không được xóa phần resources.

---

## 6. ✏️ SỬA — `app/overlays/dev/kustomization.yaml`

Thêm `hpa.yaml` vào `resources`, và XÓA dòng patch `replicas-patch.yaml`. Kết quả đầy đủ:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ecommerce-dev
resources:
  - ../../base
  - namespace.yaml
  - hpa.yaml
  # - backend-sealed-secret.yaml
patches:
  - target: { kind: Ingress, name: ecommerce-ingress }
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: dev.k8s.nguyendat.tech
```

## 7. ✏️ SỬA — `app/overlays/staging/kustomization.yaml`

Tương tự dev, chỉ khác namespace và host:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ecommerce-staging
resources:
  - ../../base
  - namespace.yaml
  - hpa.yaml
  # - backend-sealed-secret.yaml
patches:
  - target: { kind: Ingress, name: ecommerce-ingress }
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: staging.k8s.nguyendat.tech
```

## 8. ✏️ SỬA — `app/overlays/prod/kustomization.yaml`

Thêm `hpa.yaml`, đổi patch từ `replicas-patch.yaml` thành `resources-patch.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ecommerce
resources:
  - ../../base
  - namespace.yaml
  - hpa.yaml
  - backend-sealed-secret.yaml
patches:
  - path: resources-patch.yaml
  - target: { kind: Ingress, name: ecommerce-ingress }
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: k8s.nguyendat.tech
```

---

## 9. ✏️ SỬA — `argocd/dev.yaml`, `argocd/staging.yaml`, `argocd/prod.yaml`

Thêm khối `ignoreDifferences` vào cuối `spec` của CẢ BA file (giữ nguyên mọi phần khác:
`source`, `destination`, `syncPolicy`). Khối cần thêm:

```yaml
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

Ví dụ `argocd/prod.yaml` sau khi sửa sẽ có dạng:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <giữ nguyên URL hiện tại>
    targetRevision: main
    path: app/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: ecommerce
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

---

## 10. ❌ XÓA

```
app/overlays/dev/replicas-patch.yaml
app/overlays/staging/replicas-patch.yaml
app/overlays/prod/replicas-patch.yaml
```

---

## Kiểm tra sau khi sửa (chạy trước khi commit)

Build thử cả ba overlay, không được lỗi và phải thấy HPA + KHÔNG thấy `replicas` trong
Deployment:

```bash
kubectl kustomize app/overlays/dev
kubectl kustomize app/overlays/staging
kubectl kustomize app/overlays/prod
```

Với mỗi overlay, xác nhận:
- Có 2 object `kind: HorizontalPodAutoscaler` (backend, frontend) với đúng min/max/ngưỡng.
- Deployment KHÔNG còn field `spec.replicas`.
- Deployment CÒN `resources.requests.cpu` (bắt buộc cho HPA).
- Không còn tham chiếu `replicas-patch.yaml` ở đâu.

## Lưu ý vận hành (không thuộc phần sửa code, chỉ để biết)

- HPA cần **metrics-server** trong cluster. Kiểm tra bằng `kubectl top pods`. Thiếu nó thì
  HPA không có dữ liệu CPU và sẽ không scale.
- Sau khi commit + push, Argo CD tự áp dụng. Kiểm tra `kubectl get hpa -n <namespace>`.