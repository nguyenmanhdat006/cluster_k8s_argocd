# Triển khai Kubernetes + Argo CD bằng Ansible

Chạy **một lệnh** để dựng cụm Kubernetes 3 node (1 master + 2 worker) trên VMware, kèm
Argo CD và các add-on (metrics-server, ingress-nginx, sealed-secrets).

- `192.168.253.111` → master
- `192.168.253.112` → worker
- `192.168.253.113` → worker

---

## Bạn cần chuẩn bị gì trước

**1. Một "máy điều khiển"** để chạy Ansible — có thể là laptop của bạn, hoặc chính máy
master. Máy này cần cài Ansible:
```bash
sudo apt update && sudo apt install -y ansible
```

**2. SSH key đã copy sẵn lên cả 3 máy** (để Ansible đăng nhập không cần mật khẩu):
```bash
# nếu chưa có key thì tạo:
ssh-keygen -t rsa
# copy key lên từng máy (nhập mật khẩu 1 lần cho mỗi máy):
ssh-copy-id devops@192.168.253.111
ssh-copy-id devops@192.168.253.112
ssh-copy-id devops@192.168.253.113
```
(Đổi `devops` thành user của bạn.)

**3. Ba máy có kết nối Internet** — vì playbook tải Kubernetes và các add-on từ mạng về.

---

## Cách chạy — 3 bước

### Bước 1 — Sửa 2 dòng trong `inventory.ini`
Mở file `inventory.ini`, chỉnh cho khớp máy bạn:
```ini
ansible_user=devops                          # ← tên user SSH của bạn
ansible_ssh_private_key_file=~/.ssh/id_rsa   # ← đường dẫn private key
```
(IP đã điền sẵn theo 3 máy bạn cung cấp. Hostname để mặc định là được.)

### Bước 2 — Kiểm tra Ansible vào được 3 máy
```bash
ansible all -m ping
```
Cả 3 máy trả về `SUCCESS` màu xanh là ổn. Nếu lỗi → xem mục Xử lý sự cố bên dưới.

### Bước 3 — Chạy triển khai
```bash
ansible-playbook site.yml -K
```
- `-K` để Ansible hỏi **mật khẩu sudo** của bạn (gõ 1 lần). Nếu user của bạn có sudo
  không cần mật khẩu thì bỏ `-K`.
- Quá trình mất khoảng **10–20 phút**. Cứ để nó chạy.

Xong, ở cuối màn hình Ansible sẽ in ra **mật khẩu admin của Argo CD**.

---

## Sau khi chạy xong

**Kiểm tra cluster** (chạy trên máy master, hoặc máy nào có kubeconfig):
```bash
kubectl get nodes
# phải thấy 3 node ở trạng thái Ready

kubectl get pods -A
# các pod hệ thống + argocd đang chạy
```

**Vào giao diện Argo CD:**
```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# rồi mở trình duyệt: https://localhost:8080
# User: admin   |   Password: (dòng Ansible in ra ở cuối)
```

---

## Playbook làm những gì (để bạn biết, không cần thuộc)

| Giai đoạn | Việc |
|-----------|------|
| 1. Chuẩn bị mọi node | Tắt swap, cấu hình mạng, cài containerd + kubelet/kubeadm/kubectl |
| 2. Master | `kubeadm init`, cài Flannel (mạng pod), sinh lệnh join |
| 3. Worker | Join 2 worker vào cluster |
| 4. Add-on | metrics-server, ingress-nginx, sealed-secrets, Argo CD |

Playbook **chạy lại được nhiều lần** không hỏng (idempotent) — lỡ lỗi giữa chừng cứ chạy lại.

---

## ⚠️ Lưu ý quan trọng

**1. Đây là cluster MỚI → sealed-secrets sinh khóa MỚI.**
SealedSecret cũ của bạn (đã seal trên cluster cũ) sẽ **không giải mã được** trên cluster mới,
vì mỗi controller có khóa riêng. Sau khi dựng xong, bạn phải **seal lại secret** cho cluster
này (dùng `kubeseal` trỏ tới controller mới) rồi commit bản mới vào repo GitOps.

**2. ingress-nginx ở đây dùng NodePort** (vì là máy bare-metal/VM, không có LoadBalancer
sẵn). Truy cập ingress qua `http://<IP-node>:<nodeport>`. Xem port:
```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

**3. Chỉ 1 master** → nếu master hỏng là mất quyền điều khiển cluster. Chấp nhận được cho
môi trường học/lab; production cần nhiều master (HA).

**4. Triển khai app GitOps:** sau khi cluster chạy, đưa app của bạn vào bằng cách apply
Application của repo GitOps (ví dụ `kubectl apply -f argocd/prod.yaml` từ repo gitops-demo),
hoặc điền `gitops_bootstrap_manifest` trong `group_vars/all.yml` để playbook tự áp.

---

## Xử lý sự cố

**`ansible all -m ping` báo lỗi SSH:**
- Sai user → sửa `ansible_user` trong `inventory.ini`.
- Chưa copy key → chạy lại `ssh-copy-id` ở trên.
- Sai đường dẫn key → sửa `ansible_ssh_private_key_file`.

**Playbook dừng ở bước sudo:** thêm `-K` để nhập mật khẩu sudo.

**Node kẹt ở `NotReady`:** thường do mạng (Flannel) chưa lên. Đợi 1–2 phút rồi
`kubectl get pods -n kube-flannel` xem pod flannel đã Running chưa.

**Muốn làm lại từ đầu một máy:** trên máy đó chạy `sudo kubeadm reset -f` rồi chạy lại playbook.
