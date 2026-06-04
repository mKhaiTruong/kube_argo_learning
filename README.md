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

## HPA (Horizontal Pod Autoscaler)
 
HPA tự động scale số lượng Pod dựa trên CPU hoặc memory usage. Thay vì manually tăng replicas, HPA theo dõi metrics và tự quyết định.

HPA chỉ scale Pod — Deployment và ReplicaSet vẫn là 1 object duy nhất, chỉ số Pod thay đổi. Khi CPU vượt ngưỡng thì scale up, khi traffic giảm thì tự scale down về minReplicas.

---

## Ingress & Ingress Controller
 
Ingress là object định nghĩa routing rules — cho phép dùng domain thật, port 80/443, và route nhiều service qua 1 entrypoint duy nhất thay vì dùng NodePort.

```
rules:
  - host: myapp.com
    http:
      paths:
        - path: /api
          backend: grade-submission-api:3000
        - path: /
          backend: grade-submission-portal:5001
```

Ingress Controller là thằng thực thi các rules đó — Ingress chỉ là config, Controller mới là phần thật sự xử lý traffic. K8s không có sẵn Ingress Controller, phải tự cài. Phổ biến nhất là nginx-ingress

Trên cloud thật, Ingress Controller được expose qua LoadBalancer và tự assign IP public. Trên minikube/WSL thì EXTERNAL-IP sẽ <pending> — dùng port-forward để access local.

---

## RBAC (Role-Based Access Control)

Hệ thống phân quyền trong k8s — quyết định "ai được làm gì với resource nào". Mặc định Pod dùng ServiceAccount default, không có quyền gì cả.

```
ServiceAccount   >  identity (của Pod hoặc client)
Role/ClusterRole >  danh sách quyền được làm
RoleBinding      >  gắn ServiceAccount với Role
```
Role là namespace-scoped, ClusterRole là cluster-wide.

#### NOTE: Nguyên tắc quan trọng nhất: chỉ cấp đúng quyền cần thiết, không hơn. Không dùng cluster-admin trong production.

Section-nine có 2 ServiceAccount với vai trò khác nhau:
- grade-submission-api-proxy — identity của Pod, cần quyền gọi tokenreviews API để proxy verify token từ request đến.
- grade-service-account — identity của client, cần quyền gọi các API endpoints của app.


*kube-rbac-proxy là sidecar container đứng trước app, enforce RBAC trên từng request:*
```
Request + token
        ↓
proxy verify token qua tokenreviews API
        ↓ có quyền
app :3000
        ↓ không có quyền
403 Forbidden
```
Proxy cần chạy với ServiceAccount có quyền tokenreviews — nên Deployment phải khai báo serviceAccountName: grade-submission-api-proxy.

---

## Tools
 
- **kubectl** — CLI để interact với cluster
- **minikube** — chạy k8s local
- **ArgoCD** — GitOps CD tool, tự động sync từ git repo xuống cluster