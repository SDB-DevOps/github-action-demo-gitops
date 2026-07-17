# 用 ArgoCD + Argo Rollouts 实现渐进式部署

本文档记录本仓库(`github-action-demo-gitops`)从 **纯 ArgoCD 部署** 演进到
**ArgoCD + Argo Rollouts 多种部署策略** 的完整过程,包含每种策略的配置、操作、
回滚方式,以及实际落地时踩过的坑。

---

## 目录

- [1. 整体架构](#1-整体架构)
- [2. 前置环境](#2-前置环境)
- [3. 阶段一:纯 ArgoCD 部署(Deployment)](#3-阶段一纯-argocd-部署deployment)
- [4. 阶段二:引入 Argo Rollouts(Deployment → Rollout)](#4-阶段二引入-argo-rolloutsdeployment--rollout)
- [5. 策略 A:基础 Canary(按副本比例)](#5-策略-a基础-canary按副本比例)
- [6. 策略 B:Canary + Traefik 精确流量切分](#6-策略-bcanary--traefik-精确流量切分)
- [7. 策略 C:Canary + Prometheus 自动分析](#7-策略-ccanary--prometheus-自动分析)
- [8. 策略 D:蓝绿(Blue-Green)](#8-策略-d蓝绿blue-green)
- [9. 回滚速查](#9-回滚速查)
- [10. 踩坑记录](#10-踩坑记录)
- [11. 策略之间如何切换](#11-策略之间如何切换)

---

## 1. 整体架构

```
push 到应用仓库
      │  CI 构建并推送 ghcr.io/…/github-action-demo:sha-<short>
      ▼
Deploy workflow 修改本仓库 kustomization.yaml 的 newTag → 开 PR
      │  (人工 review & merge)
      ▼
merge 到 main
      │
      ▼
ArgoCD 检测到 git 变化 → 同步进集群
      │
      ▼
Argo Rollouts 控制器接管 Rollout,按所选策略渐进式发布
      │
      ▼
外部流量:Traefik(K3s 自带) → IngressRoute → Service → Pod
```

核心分工:

| 组件 | 职责 |
|---|---|
| **ArgoCD** | GitOps 引擎。git 是唯一真相源,自动把期望状态同步进集群 |
| **Argo Rollouts** | 替代原生 Deployment,提供 canary / blue-green 等渐进式发布策略 |
| **Traefik** | K3s 自带网关,负责外部流量入口;canary 精确切流时做加权路由 |
| **Prometheus** | 提供指标,让 canary 能根据成功率**自动**放量/回滚 |

---

## 2. 前置环境

- **K3s** 单节点集群(自带 Traefik v3,`traefik.io/v1alpha1`)。
- **ArgoCD**:已安装,`argocd/application.yaml` 定义了监听 `apps/github-action-demo` 的 Application,开启了 `automated.prune` 和 `selfHeal`。
- **Argo Rollouts** 控制器(v1.9.0)+ kubectl 插件:
  ```bash
  # Linux 二进制安装 kubectl 插件
  curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
  chmod +x kubectl-argo-rollouts-linux-amd64
  sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
  kubectl argo rollouts version
  ```
- **Prometheus**(仅策略 C 需要):见 `infra/prometheus/prometheus.yaml`(精简单 Pod 版,适配小内存节点)。

> ⚠️ ArgoCD 需要能识别 `Rollout` 的健康状态,确保 argocd 已内置或配置了 Argo Rollouts 的
> health check(否则 Application 会一直停在 Progressing)。

---

## 3. 阶段一:纯 ArgoCD 部署(Deployment)

最初只用 ArgoCD + 原生 `Deployment`,实现 GitOps 基础流程。

**关键文件**(演进后已被 Rollout 取代,此处仅说明原理):

```yaml
# deployment.yaml(原始版本)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-action-demo
spec:
  replicas: 1
  selector:
    matchLabels: { app: github-action-demo }
  template:
    ...
```

```yaml
# kustomization.yaml —— newTag 是"部署什么"的唯一真相源
images:
  - name: ghcr.io/sdb-devops/github-action-demo
    newTag: sha-xxxxxxx
```

**发布流程**:CI 改 `newTag` → PR → merge → ArgoCD 同步 → Deployment 滚动更新。

**局限**:原生 Deployment 的 `RollingUpdate` 无法做「先给 25% 流量观察一下」、
「指标不对自动回滚」、「一键蓝绿切换」这类渐进式发布。→ 引入 Argo Rollouts。

---

## 4. 阶段二:引入 Argo Rollouts(Deployment → Rollout)

把 `Deployment` 换成 `Rollout`。Pod 模板部分几乎一模一样,只是:

- `apiVersion: argoproj.io/v1alpha1`,`kind: Rollout`
- 多了 `spec.strategy`(canary 或 blueGreen)

kustomization 的 `images:` 转换器对 Rollout 同样生效,所以 **CI 改 tag 的流程完全不变**。

```yaml
# kustomization.yaml
resources:
  - rollout.yaml      # 原来是 deployment.yaml
  - service.yaml
  - ...
```

> Argo Rollouts 用 `rollouts-pod-template-hash` 标签区分每个版本的 ReplicaSet,
> 并**运行时把这个标签注入到 Service 的 selector**,从而把不同 Service 指向不同版本。
> 这是后面所有策略的底层机制。

---

## 5. 策略 A:基础 Canary(按副本比例)

最简单的 canary,**不需要任何网关集成**,靠副本比例近似地分流。

```yaml
# rollout.yaml
spec:
  replicas: 4          # 权重要能整除副本:25%→1, 50%→2, 75%→3
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause: {}                    # 无限期暂停,等人工 promote
        - setWeight: 50
        - pause: { duration: 30s }
        - setWeight: 75
        - pause: { duration: 30s }
```

**原理**:`setWeight: 25` = 让 25% 的 **pod** 跑新版本,Service 把流量**大致均分**给所有 pod。

**优点**:零依赖,原样 Service 即可。
**缺点**:流量比例不精确(受副本数限制、连接复用影响);粒度受副本数约束。

**操作**:
```bash
kubectl argo rollouts get rollout github-action-demo --watch   # 观察,会停在 paused
kubectl argo rollouts promote github-action-demo               # 继续下一步
kubectl argo rollouts abort   github-action-demo               # 回滚到 stable
```

---

## 6. 策略 B:Canary + Traefik 精确流量切分

要做到**按请求百分比**精确切流(而非按 pod 比例),接入 Traefik 的 traffic routing。

### 涉及的资源

```
IngressRoute → TraefikService(加权) ├─ github-action-demo-stable  → 旧版 pod
                                     └─ github-action-demo-canary  → 新版 pod
```

- **`service.yaml`**:拆成 `-stable` / `-canary` 两个 ClusterIP Service(selector 由控制器注入 hash)。
- **`traefikservice.yaml`**:`TraefikService`,控制器在每一步动态改两个后端的 `weight`。
- **`ingressroute.yaml`**:`IngressRoute` 指向上面的 `TraefikService`。
- **`rollout.yaml`**:strategy 增加 traffic routing。

```yaml
# rollout.yaml
spec:
  replicas: 2                          # 精确切流后,副本数不再决定流量比例
  strategy:
    canary:
      canaryService: github-action-demo-canary
      stableService: github-action-demo-stable
      trafficRouting:
        traefik:
          weightedTraefikServiceName: github-action-demo
      steps:
        - setWeight: 25
        - pause: {}
        - setWeight: 50
        - pause: { duration: 30s }
        - setWeight: 75
        - pause: { duration: 30s }
```

```yaml
# traefikservice.yaml
apiVersion: traefik.io/v1alpha1
kind: TraefikService
metadata:
  name: github-action-demo
spec:
  weighted:
    services:
      - { name: github-action-demo-stable, port: 80, weight: 100 }
      - { name: github-action-demo-canary, port: 80, weight: 0 }
```

### ⚠️ 关键坑:Traefik API group

Argo Rollouts v1.9.0 **默认用老的 `traefik.containo.us`** 去操作 TraefikService,
而 K3s 的 Traefik v3 只有 `traefik.io`,导致控制器一直报
`the server could not find the requested resource`,canary 卡住扩不起来。

**修复**:给控制器加命令行参数(ConfigMap 里的 `traefikAPIVersion` 在 v1.9.0 不生效):

```bash
kubectl -n argo-rollouts patch deploy argo-rollouts --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args","value":[
    "--traefik-api-group=traefik.io",
    "--traefik-api-version=traefik.io/v1alpha1"
  ]}
]'
```

用 `kubectl apply -f install.yaml` 装的,这个改动会被覆盖 → 建议用 kustomize overlay 持久化:

```yaml
# infra/argo-rollouts/kustomization.yaml
resources:
  - https://github.com/argoproj/argo-rollouts/releases/download/v1.9.0/install.yaml
patches:
  - target: { kind: Deployment, name: argo-rollouts }
    patch: |
      - op: add
        path: /spec/template/spec/containers/0/args
        value:
          - --traefik-api-group=traefik.io
          - --traefik-api-version=traefik.io/v1alpha1
```

### canary 期间 / 完成后的 Service 指向

| 时刻 | stable Service | canary Service | 权重 |
|---|---|---|---|
| 发布中 | 旧版 hash | 新版 hash | 逐步 25/50/75 |
| 完成后 | **新版 hash** | 新版 hash | 100 / 0 |

> canary 后端没起来前,canary Service 会**暂时指向 stable**,避免流量打到空服务(502)。
> 完成后 stable Service 被切到新版,旧 RS 缩容保留(供回滚)。

---

## 7. 策略 C:Canary + Prometheus 自动分析

在策略 B 的基础上,让 canary **根据 Prometheus 指标自动放量/回滚**,无需手动 promote。

### 前置:精简 Prometheus

节点内存小(3.8Gi),**不要用 kube-prometheus-stack 全家桶**(会 OOM),用单 Pod 精简版:

- 见 `infra/prometheus/prometheus.yaml`:单 Deployment + ConfigMap(抓取配置)+ Service。
- 抓取目标是 Traefik 的 metrics:`traefik-metrics.kube-system.svc:9100`。
- retention 1 天、内存限 400Mi。

启用 K3s Traefik 的 metrics(通过 `HelmChartConfig`):

```yaml
# kube-system 里
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata: { name: traefik, namespace: kube-system }
spec:
  valuesContent: |-
    metrics:
      prometheus:
        service: { enabled: true }
```
> 生效后会出现 `traefik-metrics` service(ClusterIP:9100)。

### AnalysisTemplate

```yaml
# analysistemplate.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - { name: prometheus-addr, value: http://prometheus.monitoring.svc:9090 }
  metrics:
    - name: canary-success-rate
      interval: 30s
      count: 5
      successCondition: result[0] >= 0.95    # 成功率 ≥95% 才过
      failureLimit: 2                          # 累计 2 次失败 → 自动回滚
      provider:
        prometheus:
          address: "{{args.prometheus-addr}}"
          query: |
            (
              sum(rate(traefik_service_requests_total{
                service="default-github-action-demo-canary-80@kubernetes", code=~"2.."}[2m]))
              /
              sum(rate(traefik_service_requests_total{
                service="default-github-action-demo-canary-80@kubernetes"}[2m]))
            ) or on() vector(1)      # 无数据兜底:视为健康,避免冷启动误杀
```

### 接进 Rollout(把 pause 换成 analysis)

```yaml
# rollout.yaml strategy.canary.steps
steps:
  - setWeight: 25
  - analysis:
      templates: [{ templateName: success-rate }]
  - setWeight: 50
  - analysis:
      templates: [{ templateName: success-rate }]
  - setWeight: 75
  - pause: { duration: 60s }
```

**行为**:25% → 自动查 5 次成功率 → 达标进 50% → 再查 → 进 75% → 全量;
任一分析累计失败 2 次 → **自动 abort 回滚**。

### 注意事项

1. **必须有流量**:`rate()` 靠请求量。canary 期间要持续打流量,否则测不出成功率:
   ```bash
   while true; do curl -s http://<traefik-ip>/ >/dev/null; sleep 0.2; done
   ```
2. **非 canary 期间只有 stable 标签有数据**(canary 权重为 0,不接流量)。
3. **PromQL 里的 `service` 标签**要在 Prometheus UI 里核对真实值(格式
   `default-github-action-demo-<stable|canary>-80@kubernetes`)。
4. **`or on() vector(1)` 兜底**:无数据时判健康。副作用是「完全没流量」也会一路放行 ——
   试验/生产要注意保证真实流量。

---

## 8. 策略 D:蓝绿(Blue-Green)

蓝绿**不按权重切流**,而是一次性把 active Service 从旧版切到新版。
**不需要 TraefikService、也不依赖 `--traefik-api-version`**。

### 涉及资源

- **`service.yaml`**:`-active`(线上流量) / `-preview`(预览新版)两个 Service。
- **`ingressroute.yaml`**:直接指向 `-active` Service(`kind: Service`)。
- **`rollout.yaml`**:strategy 换成 `blueGreen`。

```yaml
# rollout.yaml
spec:
  replicas: 2
  strategy:
    blueGreen:
      activeService: github-action-demo-active      # 线上流量
      previewService: github-action-demo-preview    # 预览 green
      autoPromotionEnabled: false                    # 关掉自动晋升 → 手动 promote
      scaleDownDelaySeconds: 30                       # 切换后旧版保留 30s,便于秒回滚
      # 蓝绿没有 steps / setWeight —— 要么全旧、要么全新
```

### 操作流程

```bash
# 部署新版后:green 全量拉起(约 2 倍 pod),active 仍指向旧版
kubectl argo rollouts get rollout github-action-demo

# 切换前用 preview 单独验新版
kubectl port-forward svc/github-action-demo-preview 8080:80

# 满意 → 一次性切流量
kubectl argo rollouts promote github-action-demo

# 有问题 → 回滚(scaleDownDelay 内旧版还在,秒切回)
kubectl argo rollouts abort github-action-demo
```

- `autoPromotionEnabled: false` → **必须手动 promote 才切换**;设 `true` 则 green ready 后自动切
  (可加 `autoPromotionSeconds: N` 缓冲)。
- `scaleDownDelaySeconds` 是**切换后旧 pod 保留时长**,不是回滚窗口。想留更久反悔时间就调大。

### Canary vs Blue-Green

| | Canary | Blue-Green |
|---|---|---|
| 流量切换 | 按比例渐进(25→50→100) | 一次性全切 |
| 资源占用 | 略高于 1 倍 | **约 2 倍**(新旧全量并存) |
| Traefik 加权 | 需要(策略 B/C) | 不需要 |
| 指标自动化 | 支持(analysis steps) | 支持(pre/postPromotionAnalysis) |
| 适合 | 小流量灰度、逐步放量 | 快速整体切换 + 秒回滚 |

---

## 9. 回滚速查

> 本仓库是 **GitOps + `selfHeal: true`**,命令式操作(`undo`)会被 ArgoCD 拽回,
> **持久回滚一律走 git**。

### Canary

| 场景 | 操作 |
|---|---|
| 发布进行中 | `kubectl argo rollouts abort github-action-demo`(秒回 stable) |
| 已完成,退回旧版 | git 改回旧 tag → sync(默认会重走一遍 steps) |
| 指标坏 | 自动 abort(已配 analysis) |
| 应急绕过 GitOps | 临时关 selfHeal → `undo` |

> `abort` 是状态操作、不改 manifest,通常不与 selfHeal 冲突;但 ArgoCD re-sync 可能重新触发发布,
> 彻底不发该版本仍需改 git。想让「回滚到近期版本」跳过 steps 秒切,配 `spec.rollbackWindow`。

### Blue-Green

| 场景 | 操作 |
|---|---|
| promote 前 | `kubectl argo rollouts abort`(green 缩掉,active 不动) |
| promote 后(GitOps) | git 改回旧 tag → sync → 再 promote 一次 |
| `scaleDownDelaySeconds` 内 | 旧 pod 还在,回滚最快 |

---

## 10. 踩坑记录

真实落地时遇到的问题,备查:

1. **Traefik API group 不匹配**(策略 B)
   - 现象:`the server could not find the requested resource`,canary 卡在 step 0、新 RS 一直 0 副本。
   - 根因:v1.9.0 默认查 `traefik.containo.us`,集群只有 `traefik.io`。
   - 修复:控制器加 `--traefik-api-group=traefik.io` / `--traefik-api-version=traefik.io/v1alpha1`
     (ConfigMap 的 `traefikAPIVersion` 在此版本无效)。

2. **kube-prometheus-stack 压垮小节点**(策略 C)
   - 现象:装完后 `kubectl` 报 `TLS handshake timeout`,`free -h` 只剩 ~282Mi。
   - 根因:3.8Gi 节点扛不住全家桶(Prometheus+Grafana+exporters)。
   - 修复:卸载,改用 `infra/prometheus/prometheus.yaml` 精简单 Pod 版。

3. **分析冷启动误判**(策略 C)
   - 现象:canary 刚起、流量还没到,成功率算不出 → 误判失败回滚。
   - 修复:PromQL 加 `or on() vector(1)` 兜底 + 试验时持续打流量。

4. **跨策略切换遗留孤儿 ReplicaSet**(canary → blue-green)
   - 现象:切到 blue-green 后,之前 canary 的 stable RS(带 AnalysisRun)仍有 2 个 pod 长期 Running,
     `Current` 比 `Desired` 多。
   - 根因:blue-green 的自动缩容只管它自己 promote 的前任,上一个策略的遗留 RS 不在管理范围。
   - 确认:该 RS 的 hash 不在任何 Service selector 里(不接流量)。
   - 修复:`kubectl scale rs <old-rs> --replicas=0`(不影响线上)。往后由 blue-green 正常缩容。

---

## 11. 策略之间如何切换

本仓库当前**生效的是 blue-green**,canary 相关配置以**注释**形式保留在各文件里
(`rollout.yaml` / `service.yaml` / `ingressroute.yaml` / `kustomization.yaml`),
方便来回切换,无需 git revert。

**切回 canary** 需同时反转 4 处注释:

1. `rollout.yaml`:注释 `blueGreen` 块,取消注释 `canary` 块。
2. `service.yaml`:注释 active/preview,取消注释 stable/canary。
3. `ingressroute.yaml`:把 service 换回 `TraefikService`。
4. `kustomization.yaml`:取消注释 `- traefikservice.yaml` 和 `- analysistemplate.yaml`。

> **更优做法(TODO)**:用 kustomize overlay 把 canary / blue-green 拆成两个 overlay,
> 切换时只改 ArgoCD Application 指向哪个目录,不用来回改注释。

切换后务必检查:
```bash
kubectl argo rollouts get rollout github-action-demo    # Strategy 是否正确
kubectl get svc -l app=github-action-demo               # Service 是否符合预期
```

---

## 附:关键命令速查

```bash
# 观察 rollout(带进度树)
kubectl argo rollouts get rollout github-action-demo --watch

# 手动晋升 / 回滚
kubectl argo rollouts promote github-action-demo [--full]
kubectl argo rollouts abort   github-action-demo

# 换镜像(试验用,注意会和 selfHeal 冲突)
kubectl argo rollouts set image github-action-demo \
  github-action-demo=ghcr.io/sdb-devops/github-action-demo:<tag>

# 看 Service 当前指向哪个版本
kubectl get svc -l app=github-action-demo \
  -o custom-columns=NAME:.metadata.name,HASH:.spec.selector.rollouts-pod-template-hash

# 分析历史
kubectl get analysisrun
kubectl describe analysisrun <name>
```
