# Kubernetes + ArgoCD Learning

---
 
## Core Concepts
 
### Pod
Đơn vị nhỏ nhất trong k8s — nơi container thực sự chạy. Pod được thiết kế là **ephemeral** (tạm thời): có thể chết và được tạo lại bất cứ lúc nào, mỗi lần tạo lại sẽ có IP mới. Vì vậy không nên hardcode IP của Pod.
 
### Service
Lớp networking đứng trước Pod, có địa chỉ cố định. Service dùng **label selector** để tìm Pod — không quan tâm Pod nào, IP nào, miễn có đúng label là route traffic vào. Nhờ đó Pod có thể chết/sống lại thoải mái mà Service vẫn luôn tìm được.

Service có 3 loại:
- `ClusterIP` — chỉ accessible trong cluster, dùng cho internal communication giữa các service
- `NodePort` — expose qua port của Node, chỉ dùng khi prototype/test
- `LoadBalancer` — dùng trên cloud (AWS/GCP/Azure)
Trong thực tế production, dùng **Ingress** thay NodePort để expose app ra ngoài.
 
### Deployment
Object quản lý Pod, đứng bên trên trong hierarchy:

```
Deployment  →  khai báo desired state ("tao muốn 3 Pod chạy")
    ↓
ReplicaSet  →  đảm bảo luôn đúng số lượng Pod
    ↓
Pod         →  nơi app thực sự chạy
```
 
Thực tế không tạo Pod trực tiếp mà tạo Deployment — Deployment tự tạo và quản lý Pod. Khi cần update, Deployment xóa Pod cũ và tạo Pod mới thay thế.
 
### Quan hệ giữa các object
 
```
User
  ↓
Service (lễ tân - địa chỉ cố định, tìm Pod qua label)
  ↓
Pod (do Deployment quản lý)
```
 
Service hoàn toàn độc lập với Deployment/ReplicaSet — chúng không nằm chung hierarchy mà phối hợp với nhau qua label.
 
---
 
## Health Checks
 
Thay vì dùng sidecar container để check health, k8s có sẵn native probes:
 
```yaml
livenessProbe:        # restart container nếu app chết
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
 
readinessProbe:       # chỉ route traffic vào khi app sẵn sàng
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 3
```
 
---
 
## Rolling Update & Rollback
 
Deployment hỗ trợ rolling update — thay thế Pod từng cái một, không có downtime:
 
```bash
# Update image
kubectl set image deployment/my-app my-app=my-app:v2
 
# Xem trạng thái rollout
kubectl rollout status deployment/my-app
 
# Rollback về version trước nếu có vấn đề
kubectl rollout undo deployment/my-app
```
 
---

## Stateless vs Stateful
 
**Stateless** — Pod không cần nhớ gì, chết rồi tạo mới là chạy tiếp. Phù hợp với frontend, API.
 
**Stateful** — Pod cần identity cố định và data persistent. Điển hình là database — không thể tạo Pod mới mà mất data cũ.
 
## Persistent Volume (PV)
 
Storage thật sự — có thể là local disk, cloud storage (AWS EBS, GCP Disk...). Tồn tại độc lập với Pod.
 
## Persistent Volume Claim (PVC)
 
"Tờ đơn xin" storage. Pod không gắn trực tiếp với PV mà thông qua PVC — nhờ đó Pod chết/sống lại vẫn reconnect được đúng storage cũ.
 
```
Pod  →  PVC  →  PV  →  storage thật
```
 
Xóa Pod hay StatefulSet không xóa PVC — data vẫn còn. Phải xóa PVC thủ công nếu muốn xóa sạch.

---

## Tools
 
- **kubectl** — CLI để interact với cluster
- **minikube** — chạy k8s local
- **ArgoCD** — GitOps CD tool, tự động sync từ git repo xuống cluster