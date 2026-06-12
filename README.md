# Ship Smartly — GitOps + Observability + Canary Auto-Abort

## Mục tiêu

Đưa bản mới API ra an toàn & tự bảo vệ bằng 3 mảng ghép thành 1 pipeline:

**GitOps** → mọi thay đổi qua Git, ArgoCD tự sync, rollback bằng `git revert < 5'`  
**Observability** → 1 SLO + 1 alert fire về email khi chất lượng tụt  
**Canary tự động** → thả dần traffic, bản tốt lên 100%, bản lỗi tự abort về bản cũ

---

## Kiến trúc

```
GitHub repo
  └── ArgoCD root (argocd/root.yaml)
        ├── kube-prometheus-stack  → Prometheus + Grafana + Alertmanager
        ├── argo-rollouts          → Argo Rollouts controller
        ├── web                    → nginx demo
        └── api                    → Flask API Rollout

Git push → ArgoCD sync → canary 25→50→75→100%
              → AnalysisTemplate đo success rate mỗi bước
                  → pass: promote | fail: auto-abort + alert email
```

---

## Thành phần

| File | Vai trò |
|---|---|
| `k8s-api/api.yaml` | Rollout canary 7 bước + Service + load pod |
| `k8s-api/analysis-template.yaml` | AnalysisTemplate đo success rate >= 95% |
| `k8s-api/slo-alert.yaml` | PrometheusRule: SLO 99.5%, fast + slow burn alert |
| `k8s-api/alertmanager-email.yaml` | AlertmanagerConfig gửi email |
| `k8s-api/servicemonitor.yaml` | Prometheus scrape `/metrics` |

---

## SLO & Query

**SLO = 99.5% availability**

**SLI query (AnalysisTemplate):**
```promql
sum(rate(flask_http_request_total{namespace="demo", job="api", status!~"5.."}[1m]))
/
clamp_min(sum(rate(flask_http_request_total{namespace="demo", job="api"}[1m])), 1)
```
- `>= 0.95` → pass | `< 0.95` → fail
- Đo 4 lần × 30s, `failureLimit=1` → 2 lần fail → auto-abort

**Alert rules:**

`ApiFastBurnRate` (critical) — error_rate_5m > 0.072 AND error_rate_1h > 0.072, `for: 2m`  
→ burn rate 14.4×, hết error budget trong ~2 giờ

`ApiSlowBurnRate` (warning) — error_rate_30m > 0.03 AND error_rate_6h > 0.03, `for: 5m`  
→ burn rate 6×, hết trong ~5 ngày

Email nhận alert: `Hoangkimhung2004@gmail.com`

---

## Demo scenarios

| Scenario | Config | Kết quả |
|---|---|---|
| Bản tốt | `ERROR_RATE=0`, `image=w9-api:2` | Analysis pass → promote 100% |
| Bản lỗi | `ERROR_RATE=0.8`, `image=w9-api:1` | Analysis fail → auto-abort → giữ stable |

---

## Đối chiếu yêu cầu chấm

| Yêu cầu | Trạng thái | Evidence |
|---|---|---|
| Thay đổi qua Git · ArgoCD Synced · reproduce từ Git | ✅ | evidence/Argo CD Applications.jpg |
| `git revert` rollback < 5 phút | ✅ | evidence/Git revert rollback.jpg |
| 1 SLO + alert fire về email khi inject lỗi | ✅ | evidence/ApiFastBurnRate đang FIRING!.jpg, evidence/Alert fire và gửi email.jpg |
| Canary bản lỗi tự abort về bản cũ | ✅ | evidence/AnalysisRun Successful.jpg |
| Repo có Rollout + AnalysisTemplate + SLO/alert qua Git | ✅ | evidence/Kubernetes resources.jpg |
