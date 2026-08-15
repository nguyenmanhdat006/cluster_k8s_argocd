# GitOps — Ứng dụng Ecommerce (3 môi trường)

Triển khai ứng dụng ecommerce lên Kubernetes theo mô hình **GitOps** với Argo CD, quản lý
ba môi trường **dev / staging / prod** bằng kỹ thuật **Kustomize base/overlays**.

Nguyên tắc cốt lõi: Git là nguồn sự thật duy nhất. Mọi thay đổi được thực hiện qua commit;
Argo CD tự động đồng bộ trạng thái cluster cho khớp với Git.

---

## Kiến trúc thư mục

```
gitops-demo/
├── argocd/                      # Các Argo CD Application (apply thủ công 1 lần)
│   ├── dev.yaml                 #   → theo dõi app/overlays/dev
│   ├── staging.yaml             #   → theo dõi app/overlays/staging
│   └── prod.yaml                #   → theo dõi app/overlays/prod
│
└── app/
    ├── base/                    # Manifest DÙNG CHUNG, trung lập môi trường
    │   ├── backend.yaml         #   Deployment + Service backend
    │   ├── frontend.yaml        #   Deployment + Service frontend
    │   ├── ingress.yaml         #   Định tuyến (host để placeholder)
    │   └── kustomization.yaml
    │
    └── overlays/                # KHÁC BIỆT theo từng môi trường
        ├── dev/
        │   ├── kustomization.yaml
        │   ├── namespace.yaml
        │   ├── replicas-patch.yaml
        │   └── backend-sealed-secret.example.yaml
        ├── staging/
        │   └── ... (tương tự)
        └── prod/
            ├── kustomization.yaml
            ├── namespace.yaml
            ├── replicas-patch.yaml
            └── backend-sealed-secret.yaml     # secret thật (đã seal)
```

## Nguyên lý base/overlays

`base/` chứa những gì **không đổi** giữa các môi trường. Mỗi `overlays/<env>/` chỉ khai
báo **phần khác biệt** của môi trường đó, rồi Kustomize ghép lại:

```
        base/                overlays/<env>/            Kết quả
   ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
   │ backend       │        │ namespace     │        │ manifest      │
   │ frontend      │  +     │ số replica    │   =    │ hoàn chỉnh    │
   │ ingress       │        │ host          │        │ cho môi       │
   │ (không ns,    │        │ tài nguyên    │        │ trường đó     │
   │  host tạm)    │        │ secret        │        │               │
   └──────────────┘        └──────────────┘        └──────────────┘
```

## Khác biệt giữa ba môi trường

| Thuộc tính        | dev                     | staging                     | prod                    |
|-------------------|-------------------------|-----------------------------|-------------------------|
| Namespace         | `ecommerce-dev`         | `ecommerce-staging`         | `ecommerce`             |
| Replicas          | 1                       | 2                           | 3                       |
| Host              | `dev.k8s.nguyendat.tech`| `staging.k8s.nguyendat.tech`| `k8s.nguyendat.tech`    |
| Tài nguyên backend| mặc định (base)         | mặc định (base)             | cấp cao hơn             |
| Sealed Secret     | cần seal cho namespace  | cần seal cho namespace      | có sẵn (seal cho `ecommerce`) |

Quy ước namespace: môi trường prod dùng namespace gốc `ecommerce`; các môi trường phi-prod
dùng hậu tố `-<env>`. Đây là một quy ước phổ biến giúp phân tách rõ ràng.

---

## Yêu cầu hệ thống

- Cluster Kubernetes đã cài **Argo CD**.
- Đã cài **sealed-secrets controller** (để giải mã `backend-secret`).
- **ingress-nginx** đang chạy (để định tuyến qua host; không bắt buộc để quan sát vòng lặp).

## Hướng dẫn triển khai

### Bước 1 — Cấu hình repo
Mở ba file trong `argocd/`, thay `REPLACE_GIT_REPO_URL` bằng URL repository của bạn:

```bash
GIT_URL="https://github.com/YOUR-ORG/gitops-demo.git"
grep -rl REPLACE_GIT_REPO_URL . | xargs sed -i "s|REPLACE_GIT_REPO_URL|$GIT_URL|g"
```

### Bước 2 — Đẩy lên Git
Argo CD đọc cấu hình từ Git, không đọc từ máy cục bộ. Commit và push toàn bộ trước.

### Bước 3 — Triển khai môi trường prod (chạy được ngay)
Môi trường prod dùng sealed secret có sẵn (đã seal cho namespace `ecommerce`):

```bash
kubectl apply -f argocd/prod.yaml
```

Kiểm tra:
```bash
kubectl get application ecommerce-prod -n argocd     # trạng thái Synced / Healthy
kubectl get pods -n ecommerce                        # 3 backend + 3 frontend
kubectl get secret backend-secret -n ecommerce       # controller đã giải mã
```

### Bước 4 — Chuẩn bị secret cho dev / staging
Mỗi môi trường cần secret riêng, seal lại cho đúng namespace (SealedSecret bị ràng buộc
với namespace). Tạo cho `ecommerce-dev`:

```bash
kubectl create secret generic backend-secret \
  --from-literal=DATABASE_URL='<gia-tri-cua-ban>' \
  --from-literal=FRONTEND_ORIGIN='<gia-tri-cua-ban>' \
  --from-literal=PORT='8080' \
  -n ecommerce-dev --dry-run=client -o yaml \
| kubeseal --format yaml \
    --controller-name sealed-secrets \
    --controller-namespace <NAMESPACE_CONTROLLER> \
> app/overlays/dev/backend-sealed-secret.yaml
```

Sau đó bỏ comment dòng `backend-sealed-secret.yaml` trong
`app/overlays/dev/kustomization.yaml`, commit, push. Làm tương tự cho staging.

### Bước 5 — Triển khai dev / staging
```bash
kubectl apply -f argocd/dev.yaml
kubectl apply -f argocd/staging.yaml
```

Hoặc triển khai cả ba môi trường cùng lúc:
```bash
kubectl apply -f argocd/
```

---

## Trải nghiệm vòng lặp GitOps

Đây là điểm cốt lõi cần nắm. Thử thay đổi cấu hình môi trường dev:

1. Mở `app/overlays/dev/replicas-patch.yaml`, đổi `replicas: 1` thành `replicas: 3`.
2. Commit và push:
   ```bash
   git add app/overlays/dev/replicas-patch.yaml
   git commit -m "scale dev backend to 3"
   git push
   ```
3. Không chạy lệnh `kubectl apply`. Chỉ quan sát:
   ```bash
   kubectl get pods -n ecommerce-dev -w
   ```

Argo CD phát hiện thay đổi trong Git và tự động cập nhật cluster. Quan trọng: thay đổi chỉ
tác động môi trường dev, prod và staging không bị ảnh hưởng — đó là giá trị của việc tách
overlays.

Kiểm chứng cơ chế tự phục hồi (self-heal):
```bash
kubectl scale deployment backend -n ecommerce-dev --replicas=1   # sửa tay
kubectl get pods -n ecommerce-dev -w                             # Argo CD kéo về đúng Git
```

---

## Vận hành thường ngày

**Thay đổi cấu hình một môi trường:** sửa file trong `overlays/<env>/`, commit, push.

**Thay đổi áp dụng cho mọi môi trường:** sửa file trong `base/`, commit, push. Cả ba môi
trường cùng nhận thay đổi.

**Thêm môi trường mới (ví dụ uat):** sao chép một overlay, sửa namespace / host / replicas,
tạo `argocd/uat.yaml`, apply.

## Vai trò từng thành phần

| Thành phần | Vai trò |
|------------|---------|
| `argocd/<env>.yaml` | Argo CD Application: theo dõi một overlay, đồng bộ vào một namespace |
| `app/base/` | Manifest gốc, dùng chung cho mọi môi trường |
| `app/overlays/<env>/kustomization.yaml` | Ghép base + khai báo khác biệt của môi trường |
| `app/overlays/<env>/replicas-patch.yaml` | Patch số replica (và tài nguyên với prod) |
| `app/overlays/<env>/namespace.yaml` | Namespace riêng của môi trường |
| `app/overlays/<env>/backend-sealed-secret.yaml` | Secret đã seal cho namespace tương ứng |

## Dọn dẹp

```bash
kubectl delete -f argocd/        # xóa các Application; prune dọn theo tài nguyên đã tạo
```

## Lưu ý bảo mật

Không commit secret dạng plaintext hay private key. Chỉ commit `SealedSecret` (đã mã hóa).
Mỗi SealedSecret gắn với một namespace cụ thể; đổi namespace phải seal lại.
