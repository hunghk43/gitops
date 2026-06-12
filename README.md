# Ship Smartly — GitOps + Observability + Canary Auto-Abort

## Kiến trúc

```
GitHub repo (gitops)
  └── ArgoCD root Application
        └── argocd/apps/
              ├── kube-prometheus-stack  → Prometheus + Grafana + Alertmanager
              ├── argo-rollouts          → Argo Rollouts controller
              ├── web                    → nginx demo app
              └── api                   → Flask API (Rollout canary)

API Rollout → expose /metrics
Prometheus  → scrape qua ServiceMonitor
AnalysisTemplate → query Prometheus mỗi 30s
Alertmanager → gửi email khi SLO vi phạm
```

## Thành phần

| File | Vai trò |
|---|---|
| `argocd/root.yaml` | App-of-apps |
| `argocd/apps/kube-prometheus-stack.yaml` | Prometheus + Grafana + Alertmanager |
| `argocd/apps/argo-rollouts.yaml` | Argo Rollouts controller |
| `argocd/apps/api.yaml` | ArgoCD Application cho api |
| `k8s-api/api.yaml` | Rollout canary 25→50→75→100%, load pod |
| `k8s-api/servicemonitor.yaml` | Prometheus scrape /metrics |
| `k8s-api/analysis-template.yaml` | AnalysisTemplate đo success rate |
| `k8s-api/slo-alert.yaml` | PrometheusRule: SLO 99.5%, fast+slow burn alert |
| `k8s-api/alertmanager-email.yaml` | AlertmanagerConfig gửi email |

## SLO & Query giải thích

### SLO định nghĩa
**SLO = 99.5% availability** → error budget = 0.5%/tháng

### SLI query (AnalysisTemplate)
```promql
sum(rate(flask_http_request_total{namespace="demo", app="api", status!~"5.."}[1m]))
/
clamp_min(sum(rate(flask_http_request_total{namespace="demo", app="api"}[1m])), 1)
```
- Tử số: số request thành công (non-5xx) trong 1 phút
- Mẫu số: tổng request, `clamp_min(..., 1)` tránh chia cho 0
- Ngưỡng: `>= 0.95` → pass | `< 0.95` → fail
- Chạy 4 lần × 30s, failureLimit=1 → 2 lần fail thì abort

### Alert rules (SLO 99.5%)

**ApiFastBurnRate** (critical):
```promql
error_rate_5m > 0.072 AND error_rate_1h > 0.072
```
- Ngưỡng 0.072 = 14.4× burn rate của SLO 99.5%
- Ý nghĩa: đang tiêu error budget nhanh gấp 14.4 lần → sẽ hết budget trong 2 giờ
- `for: 2m` → fire sau 2 phút liên tục

**ApiSlowBurnRate** (warning):
```promql
error_rate_30m > 0.03 AND error_rate_6h > 0.03
```
- Ngưỡng 0.03 = 6× burn rate
- Ý nghĩa: tiêu budget gấp 6 lần → hết trong ~5 ngày nếu không sửa
- `for: 5m`

## Kịch bản demo

| Kịch bản | Config | Kết quả |
|---|---|---|
| Bản tốt | `ERROR_RATE=0`, `image=w9-api:1` | Analysis pass, promote 25→50→75→100% |
| Bản lỗi | `ERROR_RATE=0.8`, `image=w9-api:2` | Analysis fail, auto-abort → giữ stable |

## Lệnh kiểm tra

```powershell
# ArgoCD apps
kubectl -n argocd get applications

# Tất cả resource trong demo
kubectl -n demo get rollout,pod,svc,analysistemplate,prometheusrule

# Theo dõi rollout real-time
kubectl -n demo get analysisrun -w

# Rollback qua Git (< 5 phút)
git revert HEAD
git push
# ArgoCD tự sync → Rollout về stable

# Inject lỗi để test
kubectl set env rollout/api ERROR_RATE=0.8 -n demo
```

## Đối chiếu yêu cầu chấm

| Yêu cầu | Trạng thái |
|---|---|
| Thay đổi qua Git · ArgoCD Synced · reproduce từ Git | ✅ Toàn bộ qua Git |
| git revert rollback < 5 phút | ✅ `git revert HEAD && git push` → ArgoCD sync ~2 phút |
| 1 SLO + 1 alert fire về email khi inject lỗi | ✅ `ApiFastBurnRate` → email `Hoangkimhung2004@gmail.com` |
| Canary bản lỗi tự abort về bản cũ | ✅ AnalysisTemplate đo 4×30s, 2 fail → auto-abort |
