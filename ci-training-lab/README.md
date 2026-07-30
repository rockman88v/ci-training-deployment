# ci-training-lab — Helm Chart

Helm chart để triển khai **ci-training-lab** (Node.js/Express REST API) lên Kubernetes, bao gồm Deployment, Service, Ingress, ConfigMap và Secret.

## Yêu cầu

| Công cụ | Phiên bản tối thiểu |
|---|---|
| Kubernetes | 1.24+ |
| Helm | 3.10+ |
| Ingress controller | nginx-ingress hoặc tương đương |

---

## Cài đặt nhanh

```bash
# 1. Clone repo (hoặc cd vào thư mục workspace)
cd ci-training-lab-vietdevops

# 2. Tạo imagePullSecret
kubectl create secret docker-registry ghcr-secret \
  --namespace ci-training-lab \
  --docker-server=ghcr.io \
  --docker-username=[your-github-user] \
  --docker-password='[yourtoken]' \
  --docker-email='[your-email]'

# 3. Cài đặt với giá trị tối thiểu cần thiết
helm install ci-training-lab ./ci-training-lab \
  --namespace app \
  --create-namespace \
  --set image.repository=ghcr.io/<your-org>/ci-training-lab \
  --set image.tag=<git-sha> \
  --set secret.apiKey=<your-api-key> \
  --set ingress.host=api.yourdomain.com
```

---

## Cấu hình Values

### Image

```yaml
image:
  repository: ghcr.io/your-org/ci-training-lab  # Registry + repo path
  tag: "latest"                                   # Tag được CI cập nhật tự động
  pullPolicy: IfNotPresent
```

### Số lượng replicas

```yaml
replicaCount: 2   # Tăng lên khi cần scale
```

### Service

```yaml
service:
  type: ClusterIP   # ClusterIP (mặc định) | NodePort | LoadBalancer
  port: 80          # Port của Service
  targetPort: 3000  # Port mà container lắng nghe
```

### Ingress

Ingress được bật mặc định với `ingressClassName: nginx`.

```yaml
ingress:
  enabled: true
  className: "nginx"
  host: ci-training-lab.example.com   # Thay bằng domain thực
  path: /
  pathType: Prefix
  annotations: {}
  tls: []
```

**Ví dụ bật TLS với cert-manager:**

```yaml
ingress:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  tls:
    - secretName: ci-training-lab-tls
      hosts:
        - api.yourdomain.com
```

### Environment Variables

Biến môi trường không nhạy cảm được lưu trong ConfigMap:

```yaml
env:
  NODE_ENV: production
  PORT: "3000"
```

### Secret (API_KEY)

> **Không bao giờ** commit giá trị thật vào `values.yaml`.

**Cách 1 — truyền qua `--set` khi install:**
```bash
helm install ci-training-lab ./deploy --set secret.apiKey=<your-key>
```

**Cách 2 — dùng file values riêng (không commit):**
```bash
# values-secret.yaml (thêm vào .gitignore)
secret:
  apiKey: "your-real-api-key"

helm install ci-training-lab ./deploy -f values-secret.yaml
```

**Cách 3 (khuyến nghị production) — External Secrets Operator:**  
Tạo `ExternalSecret` trỏ vào Vault / AWS Secrets Manager / GCP Secret Manager thay vì dùng `secret.apiKey` trong chart.

### Resources

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

### Liveness & Readiness Probe

Cả hai probe đều gọi endpoint `/health` của app:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 15
  periodSeconds: 20

readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## Các lệnh Helm thường dùng

```bash
# Xem trước manifest sẽ được sinh ra (không deploy)
helm template ci-training-lab ./deploy \
  --set image.tag=abc123 \
  --set ingress.host=api.example.com

# Upgrade khi có thay đổi
helm upgrade ci-training-lab ./deploy \
  --namespace app \
  --set image.tag=<new-sha>

# Xem trạng thái release
helm status ci-training-lab -n app

# Gỡ cài đặt
helm uninstall ci-training-lab -n app
```

---

## Cấu trúc chart

```
ci-training-lab/
├── Chart.yaml              # Metadata: tên, version, appVersion
├── values.yaml             # Giá trị mặc định
├── README.md               # Tài liệu này
└── templates/
    ├── _helpers.tpl        # Template helpers (name, labels, ...)
    ├── configmap.yaml      # NODE_ENV, PORT
    ├── secret.yaml         # API_KEY (Opaque Secret)
    ├── deployment.yaml     # Deployment — 2 replicas, non-root user
    ├── service.yaml        # ClusterIP Service
    └── ingress.yaml        # Ingress (nginx)
```
