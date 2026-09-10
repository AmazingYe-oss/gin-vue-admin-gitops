# gin-vue-admin GitOps 基础设施与声明式交付仓库

本项目是 [AmazingYe-oss/gin-vue-admin](https://github.com/AmazingYe-oss/gin-vue-admin) 的配套 GitOps 声明式配置仓库。作为 Kubernetes 集群部署与运维的**唯一定义事实源（Single Source of Truth）**，配合 ArgoCD 与 Helm 实现自动化交付与生产级可观测性闭环。

---

## 🏗️ 架构与组件清单

- **编排管理**：Helm v3 + k3s (Kubernetes v1.30+)
- **持续交付引擎**：ArgoCD（支持 Automated Sync、Self-Heal 自动纠偏、Prune 级联清理）
- **监控告警基础设施**：Prometheus Operator (kube-prometheus-stack)
- **配置结构**：
  ```text
  helm/gva/
  ├── Chart.yaml                  # Chart 元数据
  ├── values.yaml                 # 多环境全局变量与镜像 Tag 定义
  └── templates/
      ├── server-deployment.yaml      # 后端无状态负载 (探针/安全上下文/资源限制)
      ├── server-service.yaml         # 后端内部 ClusterIP 服务
      ├── server-config.yaml          # 动态 ConfigMap 配置文件注入
      ├── server-sqlite-pvc.yaml      # SQLite 持久化存储 (fsGroup=1000 属主解决)
      ├── server-logs.yaml            # 业务日志持久化 PVC
      ├── web-deployment.yaml         # 前端 Nginx 负载
      ├── web-service.yaml            # 前端 Service 暴露
      ├── server-servicemonitor.yaml  # ServiceMonitor 自动化指标抓取
      ├── server-alerts.yaml          # PrometheusRule 业务与存活告警规则
      └── alertmanager-config.yaml    # AlertmanagerConfig 分级路由与 Webhook 通知
  ```

---

## 🌟 GitOps 交付与核心工程设计

### 1. 参数化 Helm Chart 与平滑接管（Adoption）
- 将 10 余份手写 K8s 资源解耦并抽象为可复用 Helm Chart；
- 采用资源 Adoption 机制，在不中断线上流量的前提下，实现存量 K8s 资源向 Helm 管理的平滑平移；
- 利用 `securityContext.fsGroup: 1000` 配合非 root 镜像，优雅解决 Kubernetes 本地 PVC 挂载目录导致的 SQLite `permission denied` 权限故障。

### 2. ArgoCD 声明式交付与防漂移策略
- **Automated Sync**：监听本仓库 `main` 分支变动，CI 流水线回写镜像 Tag 后触发自动同步；
- **Self-Heal（自愈机制）**：严格拦截人工 `kubectl edit/scale` 产生的配置漂移，集群状态偏离时自动回滚至 Git 声明状态；
- **Prune（级联清理）**：Git 清单删除资源时自动清理集群孤儿对象，保持环境一致性。

### 3. Prometheus Operator 生产级可观测性
- **指标发现**：通过 `ServiceMonitor` 基于 Service 标签自动关联后端 `/metrics` 端口，实现目标动态注册；
- **双重防漏存活告警**：在 `PrometheusRule` 中配置 `up == 0`（进程异常响应）与 `absent(up) == 1`（Pod 彻底被销毁或标签丢失）互补双规则，彻底解决缩容或被驱逐时的告警盲区；
- **分级路由与告警闭环**：通过 `AlertmanagerConfig` CRD 定义告警路由树，按 `severity` 等级聚合与分发，对接 Webhook 端点实现故障 Firing 与恢复 Resolved 的全链路闭环。

---

## 🔗 相关仓库

- **应用源码与 CI 仓库**：[AmazingYe-oss/gin-vue-admin](https://github.com/AmazingYe-oss/gin-vue-admin)
