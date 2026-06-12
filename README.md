# Challenge: Ship Smartly — GitOps + Observability + Canary Auto-Abort

## Mục tiêu

Triển khai API Flask lên Kubernetes theo mô hình GitOps hoàn chỉnh. Mục tiêu chính là chứng minh 3 mảng ghép thành 1 pipeline an toàn:

- **GitOps**: mọi thay đổi qua Git, ArgoCD tự sync, rollback bằng `git revert < 5 phút`
- **Observability**: SLO đo chất lượng thực tế, alert tự fire về email khi hệ thống xấu đi
- **Canary tự động**: thả dần traffic, AnalysisTemplate tự chấm — bản tốt lên 100%, bản lỗi tự abort

## Kiến trúc

```text
GitHub repo (hunghk43/gitops)
  └── ArgoCD root Application  (argocd/root.yaml)
        └── argocd/apps/
              ├── kube-prometheus-stack   → Prometheus + Grafana + Alertmanager (namespace: monitoring)
              ├── argo-rollouts           → Argo Rollouts controller (namespace: argo-rollouts)
              ├── web                     → nginx demo app (namespace: demo)
              └── api                     → Flask API Rollout (namespace: demo)

Pipeline:
  Git push (image/config thay đổi)
    → ArgoCD detect & sync
      → Argo Rollouts khởi động canary (25% → 50% → 75% → 100%)
        → AnalysisTemplate query Prometheus mỗi bước
          ├── success rate >= 95%  → promote bước tiếp
          └── success rate < 95%   → auto-abort → giữ stable
              → PrometheusRule fire alert
                → Alertmanager gửi email Hoangkimhung2004@gmail.com
```

## Thành phần chính

| Thành phần | File | Vai trò |
|---|---|---|
| Root app | `argocd/root.yaml` | App-of-apps, quản lý tất cả Application con trong `argocd/apps/` |
| Prometheus stack | `argocd/apps/kube-prometheus-stack.yaml` | Cài Prometheus, Grafana, Alertmanager qua Helm |
| Argo Rollouts | `argocd/apps/argo-rollouts.yaml` | Cài controller xử lý kind `Rollout` |
| API app | `argocd/apps/api.yaml` | ArgoCD Application trỏ tới path `k8s-api/` |
| API workload | `k8s-api/api.yaml` | Rollout canary 7 bước + Service + load pod |
| Metrics | `k8s-api/servicemonitor.yaml` | Cho Prometheus scrape `/metrics` từ pod API |
| Canary check | `k8s-api/analysis-template.yaml` | AnalysisTemplate đo success rate >= 95% |
| SLO alert | `k8s-api/slo-alert.yaml` | PrometheusRule: fast burn + slow burn alert |
| Email config | `k8s-api/alertmanager-email.yaml` | AlertmanagerConfig gửi email khi alert fire |

## SLO & Query — Giải thích

### SLO định nghĩa

**SLO = 99.5% availability** (error budget = 0.5%/tháng ≈ 3.6 giờ downtime)

### SLI query trong AnalysisTemplate

```promql
sum(rate(flask_http_request_total{namespace="demo", app="api", status!~"5.."}[1m]))
/
clamp_min(sum(rate(flask_http_request_total{namespace="demo", app="api"}[1m])), 1)
```

| Thành phần | Giải thích |
|---|---|
| Tử số | Rate request thành công (non-5xx) trong 1 phút |
| Mẫu số | Tổng rate tất cả request |
| `clamp_min(..., 1)` | Tránh chia cho 0 khi chưa có traffic |
| `status!~"5.."` | Loại bỏ HTTP 5xx (server error) |

**Ngưỡng chấm:**
- `result[0] >= 0.95` → Successful (pass bước canary)
- `result[0] < 0.95` → Failed
- `count: 4`, `interval: 30s`, `initialDelay: 30s` → đo 4 lần, cách 30s, chờ 30s trước khi bắt đầu
- `failureLimit: 1` → 2 lần fail liên tiếp → abort canary

### Alert rules — Multi-window burn rate

Dùng 2 cửa sổ thời gian để tránh false positive (Google SRE pattern).

**ApiFastBurnRate** — severity: `critical`

```promql
(error_rate_5m > 0.072) AND (error_rate_1h > 0.072)
```

- Ngưỡng `0.072` = 14.4× burn rate của SLO 99.5%
- Ý nghĩa: đang tiêu error budget nhanh gấp 14.4 lần bình thường → sẽ hết budget trong ~2 giờ
- `for: 2m` → fire sau 2 phút liên tục

**ApiSlowBurnRate** — severity: `warning`

```promql
(error_rate_30m > 0.03) AND (error_rate_6h > 0.03)
```

- Ngưỡng `0.03` = 6× burn rate
- Ý nghĩa: đang tiêu budget gấp 6 lần → hết trong ~5 ngày nếu không xử lý
- `for: 5m`

**Email nhận alert**: `Hoangkimhung2004@gmail.com`

## Kịch bản demo

| Kịch bản | Cấu hình | Kết quả kỳ vọng |
|---|---|---|
| Bản tốt | `ERROR_RATE=0`, `image=w9-api:1` | Analysis pass 4 lần → promote 25% → 50% → 75% → 100% |
| Bản lỗi | `ERROR_RATE=0.8`, `image=w9-api:2` | Analysis fail → auto-abort → stable giữ nguyên |

Load generator tạo traffic liên tục (đã tích hợp trong `k8s-api/api.yaml`):

```yaml
# Pod load trong api.yaml
while true; do wget -qO- http://api.demo.svc.cluster.local:8080/; sleep 1; done
```

## Lệnh kiểm tra

```powershell
# Tất cả ArgoCD applications
kubectl -n argocd get applications

# Toàn bộ resource trong namespace demo
kubectl -n demo get rollout,pod,svc,endpoints,servicemonitor,analysistemplate,prometheusrule

# Theo dõi AnalysisRun real-time
kubectl -n demo get analysisrun -w

# Chi tiết một AnalysisRun
kubectl -n demo describe analysisrun <tên-run>

# Inject lỗi để test auto-abort
kubectl set env rollout/api ERROR_RATE=0.8 -n demo

# Rollback qua Git (GitOps way)
git revert HEAD
git push
```

## Rollback < 5 phút

```powershell
# Xem commit gần nhất
git log --oneline -3

# Revert commit lỗi
git revert HEAD
git push

# ArgoCD tự detect thay đổi và sync (~1-2 phút)
# Rollout quay về stable image
```

---

## Evidence

### Evidence 1 — ArgoCD Applications (Synced + Healthy)

> Chụp màn hình: `kubectl -n argocd get applications` hoặc ArgoCD UI tại `localhost:8080`
> Cần thấy: tất cả apps đều `Synced` + `Healthy`

```
<!-- TODO: bỏ ảnh evidence/01-argocd-apps.png vào đây -->
```

---

### Evidence 2 — Prometheus Targets UP

> Mở Prometheus UI: `localhost:9090` → Status → Targets
> Cần thấy: `serviceMonitor/demo/api` với 4/4 endpoints UP

```
<!-- TODO: bỏ ảnh evidence/02-prometheus-targets.png vào đây -->
```

---

### Evidence 3 — Prometheus query có dữ liệu

Chạy query trên Prometheus UI:

```promql
flask_http_request_total{namespace="demo", app="api"}
```

> Cần thấy: 4 time series trả về, mỗi pod 1 series, có giá trị tăng dần

```
<!-- TODO: bỏ ảnh evidence/03-prometheus-query.png vào đây -->
```

---

### Evidence 4 — Kubernetes resources trong namespace demo

```powershell
kubectl -n demo get rollout,pod,svc,endpoints,servicemonitor,analysistemplate,prometheusrule
```

> Cần thấy: rollout, 4 pods running + pod/load, service, analysistemplate/api-success-rate, prometheusrule/api-slo

```
<!-- TODO: bỏ ảnh evidence/04-resources-demo.png vào đây -->
```

---

### Evidence 5 — Canary rollout đang chạy (bản tốt)

```powershell
kubectl -n demo get analysisrun
kubectl -n demo describe rollout api
kubectl -n demo get rs,pod -l app=api
```

> Cần thấy: canary revision mới, stable revision cũ, bước setWeight đang ở 25%/50%/75%, AnalysisRun Running

```
<!-- TODO: bỏ ảnh evidence/05-canary-rollout.png vào đây -->
```

---

### Evidence 6 — AnalysisRun Successful (bản tốt)

```powershell
kubectl -n demo get analysisrun
kubectl -n demo describe analysisrun <tên-run>
```

> Cần thấy: Status = `Successful`, 4 measurements đều `[1]` (100% success rate), promote lên 100%

```
<!-- TODO: bỏ ảnh evidence/06-analysis-successful.png vào đây -->
```

---

### Evidence 7 — Auto-abort với bản lỗi

Inject lỗi:

```powershell
# Sửa api.yaml: ERROR_RATE=0.8, image=w9-api:2, commit & push
git commit -m "test: deploy bad version with ERROR_RATE=0.8"
git push
```

```powershell
kubectl -n demo get analysisrun -w
kubectl -n demo describe rollout api
```

> Cần thấy:
> - AnalysisRun Status = `Failed`
> - Rollout Status = `Degraded`, Message = `RolloutAborted`
> - Canary ReplicaSet ScaledDown về 0
> - Stable image vẫn giữ nguyên, 4 pods stable vẫn Running

```
<!-- TODO: bỏ ảnh evidence/07-auto-abort.png vào đây -->
```

---

### Evidence 8 — PrometheusRule đã apply

```powershell
kubectl -n demo get prometheusrule api-slo -o yaml
```

> Cần thấy: 2 alert rules `ApiFastBurnRate` và `ApiSlowBurnRate` với đúng expr và ngưỡng

```
<!-- TODO: bỏ ảnh evidence/08-prometheusrule.png vào đây -->
```

---

### Evidence 9 — Alert firing trên Prometheus UI

> Mở Prometheus UI: `localhost:9090` → Alerts
> Sau khi inject ERROR_RATE=0.8, cần thấy `ApiFastBurnRate` chuyển sang `FIRING`

```
<!-- TODO: bỏ ảnh evidence/09-alert-firing.png vào đây -->
```

---

### Evidence 10 — Git revert rollback < 5 phút

```powershell
Get-Date   # ghi lại thời điểm bắt đầu
git revert HEAD
git push
Get-Date   # ghi lại thời điểm sau khi push

# Đợi ArgoCD sync (~1-2 phút)
kubectl -n argocd get applications
kubectl -n demo describe rollout api
Get-Date   # ghi lại thời điểm Healthy
```

> Cần thấy: timestamp từ lúc revert đến lúc Rollout Healthy < 5 phút, image quay về bản stable

```
<!-- TODO: bỏ ảnh evidence/10-git-revert-rollback.png vào đây -->
```

---

### Evidence 11 — Email alert nhận được

> Mở Gmail: `Hoangkimhung2004@gmail.com`
> Cần thấy: email với subject `[FIRING] ApiFastBurnRate` hoặc tương tự từ Alertmanager

```
<!-- TODO: bỏ ảnh evidence/11-alert-email.png vào đây -->
```

---

## Đối chiếu yêu cầu chấm

| Yêu cầu | Trạng thái | Evidence |
|---|---|---|
| Thay đổi qua Git · ArgoCD Synced (no drift) · reproduce được từ Git | ✅ | Evidence 1, 4 |
| `git revert` rollback < 5 phút | ✅ | Evidence 10 |
| 1 SLO + 1 alert fire về email cá nhân khi inject lỗi | ✅ | Evidence 8, 9, 11 |
| Canary bản lỗi tự abort về bản cũ (quan trọng nhất) | ✅ | Evidence 7 |
| Repo có Rollout + AnalysisTemplate + SLO/alert tất cả qua Git | ✅ | Evidence 4, 8 |
| README giải thích query & ngưỡng | ✅ | File này |
