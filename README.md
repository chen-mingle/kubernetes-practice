# kubernetes-practice

## 📖 项目简介
本项目为个人的 K8s 实践项目。基于本地 Linux 虚拟机搭建轻量级 Kubernetes (k3s) 集群，完成 Nginx 应用的容器化部署、服务暴露、滚动更新及多场景故障排查。旨在深入理解 K8s 工作负载生命周期管理与云原生运维排错思路。

## 💻 环境说明
*   **操作系统**：Ubuntu 22.04 LTS (虚拟机，分配 4C / 4G)
*   **K8s 发行版**：k3s (轻量级、CNCF 认证)
*   **kubectl 版本**：v1.28+

## 🛠️ 核心实践内容

### 1. 基础部署与流量控制
*   编写 `my-app.yaml`，使用 **Deployment** 管理 3 个 Nginx 副本。
*   配置 **readinessProbe (就绪探针)**，通过 HTTP GET 请求检测容器状态，确保只有准备就绪的 Pod 才会接收流量。
*   编写 `my-svc.yaml`，创建 **NodePort 类型的 Service**，通过标签选择器 (Label Selector) 关联后端的 Pod，对外暴露 30080 端口。

### 2. 滚动更新与版本回滚
*   **零停机发布**：配置滚动更新策略，测试 `maxSurge` 与 `maxUnavailable` 参数。观察到发布过程中旧 Pod 逐步被新 Pod 替换，业务未中断。
*   **版本回滚**：模拟发布错误镜像导致更新失败，使用 `kubectl rollout history` 查看历史，通过 `kubectl rollout undo` 成功回滚至上一稳定版本，验证了副本恢复过程。

### 3. 故障模拟与排查复盘
详细记录了实践过程中遇到的 4 个典型故障及排查过程（详见下方“排错记录”）。

---

## 🚑 故障排查记录 (Troubleshooting)

本人在实操过程中记录并解决了以下问题，沉淀了排查思路：

| 故障现象 | 排查命令 | 根因定位 (Root Cause) | 修复动作 (Resolution) |
| :--- | :--- | :--- | :--- |
| **Pod 状态卡在 `ImagePullBackOff`** | `kubectl describe pod <pod_name>` | 查看 Events 发现拉取 Docker Hub 镜像超时，导致镜像下载失败。 | 将 YAML 文件中的 `image` 替换为阿里云公共镜像源地址。 |
| **Pod 状态 `0/1 Running`，Service 无法访问** | `kubectl get endpoints`<br>`kubectl describe pod <pod_name>` | 故意配置错误的就绪探针路径 (`/abc`)，导致探针返回 404 失败，Pod 未 Ready，Endpoint 被移除。 | 修正 `readinessProbe` 路径为 `/`，重新 apply，Pod 恢复正常接收流量。 |
| **Pod 被误删后服务仍正常** | `kubectl get pods -w` | （自愈机制验证）手动 `delete pod` 后，Deployment 控制器检测到实际副本数少于期望值。 | 无需人工干预，ReplicaSet 在 3 秒内自动拉起全新的 Pod，理解 K8s 的期望状态 (Desired State)。 |
| **kubectl 执行时报权限拒绝** | `ls -l /etc/rancher/k3s/k3s.yaml` | k3s 默认配置文件权限为 `600`，普通用户无法读取。 | 未直接修改原文件权限（避免安全隐患），而是将配置复制到 `~/.kube/config` 并修改所有权，通过环境变量 `KUBECONFIG` 指定配置路径。 |

---

## 🚀 常用运维命令速查 (Cheat Sheet)
```bash
# 查看 Pod 详细事件（排查首选）
kubectl describe pod <pod_name>

# 查看容器标准输出日志（排查应用报错）
kubectl logs <pod_name> -f

# 查看发布状态
kubectl rollout status deployment/my-web

# 查看历史版本并回滚
kubectl rollout history deployment/my-web
kubectl rollout undo deployment/my-web
