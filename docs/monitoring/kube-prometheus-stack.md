# 使用 kube-prometheus-stack 搭建监控系统

## 概述

[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) 是 Prometheus 生态中用于在 Kubernetes 部署 Prometheus 相关组件的 helm chart，涵盖 Prometheus Operator、Prometheus、Alertmanager、Grafana、node-exporter、kube-state-metrics 以及社区提供的各种 Grafana 面板等，本文介绍如何使用这个 chart 在 TKE 集群中搭建监控系统。

## 安装

添加 helm repo：

```bash
helm repo add prom https://prometheus-community.github.io/helm-charts
helm repo update
```

安装：

```bash
helm upgrade --install kube-prometheus-stack prom/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --version <chart-version> \
  -f image-values.yaml \
  -f grafana-values.yaml
```

:::tip[选择合适的 chart 版本]

`kube-prometheus-stack` 的 chart 版本与 app 版本（Prometheus Operator 版本）一一对应。不同版本之间的 CRD、默认配置、镜像版本可能有较大差异。

选择 chart 版本后，`image-values.yaml` 中的镜像 tag 必须与 chart 对应的 app 版本一致，否则可能出现兼容性问题。

:::

## 自定义配置的方法

`kube-prometheus-stack` chart 非常庞大，配置项极多。建议将自定义配置拆分为多个 `values.yaml` 文件分别维护，安装时指定多个 `-f` 参数：

- `image-values.yaml`：镜像替换配置
- `grafana-values.yaml`：Grafana 和其它自定义配置

```bash
helm upgrade --install kube-prometheus-stack prom/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f image-values.yaml \
  -f grafana-values.yaml
```

如果使用 kustomize 管理，可以用 `additionalValuesFiles`：

```yaml title="kustomization.yaml"
helmCharts:
- repo: https://prometheus-community.github.io/helm-charts
  name: kube-prometheus-stack
  releaseName: kube-prometheus-stack
  namespace: monitoring
  includeCRDs: true
  version: "80.14.4"
  additionalValuesFiles:
  - image-values.yaml
  - grafana-values.yaml
```

## 国内环境替换镜像地址

`kube-prometheus-stack` 依赖的镜像主要来自 `quay.io`，国内拉取可能失败或超时。有两种解决方案：

### 使用 TKE 内网 mirror（推荐）

TKE 提供了 `quay.tencentcloudcr.com` 作为 `quay.io` 的内网 mirror，将镜像 registry 替换即可：

```yaml title="image-values.yaml"
grafana:
  sidecar:
    image:
      registry: quay.tencentcloudcr.com
alertmanager:
  alertmanagerSpec:
    image:
      registry: quay.tencentcloudcr.com
prometheus:
  prometheusSpec:
    image:
      registry: quay.tencentcloudcr.com
prometheusOperator:
  image:
    registry: quay.tencentcloudcr.com
  prometheusConfigReloader:
    image:
      registry: quay.tencentcloudcr.com
  thanosImage:
    registry: quay.tencentcloudcr.com
kube-state-metrics:
  image:
    registry: docker.io
    repository: k8smirror/kube-state-metrics
  kubeRBACProxy:
    image:
      registry: quay.tencentcloudcr.com
prometheus-node-exporter:
  image:
    registry: quay.tencentcloudcr.com
  kubeRBACProxy:
    image:
      registry: quay.tencentcloudcr.com
```

部分不在 `quay.io` 上的镜像可替换为 DockerHub 上的社区 mirror：

| 原始镜像                                              | DockerHub mirror 镜像                  |
| :---------------------------------------------------- | :------------------------------------- |
| registry.k8s.io/kube-state-metrics/kube-state-metrics | docker.io/k8smirror/kube-state-metrics |

:::tip[验证镜像在集群内是否可达]

不同集群节点的镜像加速配置不同（如 TKE 节点通常将 `docker.io` 指向内网加速器 `mirror.ccs.tencentyun.com`）。选定 mirror 镜像后，建议先用一个临时 Pod 验证节点能否拉取，避免安装时才发现镜像不可达：

```bash
kubectl run img-test --image=docker.io/xxx/yyy:tag --restart=Never --command -- true
kubectl describe pod img-test | grep -A5 Events   # 看到 "Successfully pulled" 即可达
kubectl delete pod img-test
```

只要镜像**确实存在于 DockerHub**，通过内网加速器即可拉取；若镜像在 DockerHub 上不存在，加速器回源会超时（`i/o timeout`）而非快速报错，容易误判。

:::

## 配置 Grafana

grafana 是 `kube-prometheus-stack` 的一个 subchart，所有 grafana 配置放在 `grafana` 字段下：

```yaml title="grafana-values.yaml"
grafana:
  adminUser: "roc"
  adminPassword: "<your-password>"
  defaultDashboardsTimezone: "Asia/Shanghai"
  sidecar:
    dashboards:
      folderAnnotation: "grafana_folder"
      provider:
        foldersFromFilesStructure: true
  testFramework:
    enabled: false
```

具体配置建议参考 [在 TKE 上自建 Grafana](./grafana)。

## Admission Webhooks 配置

Prometheus Operator 默认启用 admission webhooks，用于校验 `PrometheusRule` 等 CRD 的语法（例如提前拦截错误的告警规则表达式）。webhook 的 TLS 证书由 chart 自带的 certgen job 自动生成到 `kube-prometheus-stack-admission` Secret，供 operator 挂载使用。

**是否需要禁用 webhook，取决于集群的网络模式**：

- **Cilium Native（VPC-CNI）模式**：Pod IP 是真实 VPC IP，apiserver 可直接路由到 Pod，webhook 能正常工作，**无需禁用**，只需处理 certgen 镜像即可（见下文）。
- **Cilium Overlay 模式**：apiserver 运行在管控面（无 cilium-agent），无法路由到 overlay Pod IP（如 `10.244.x.x`），apiserver 调用 webhook 会连接超时，**建议禁用**（见下文）。详见 [安装 Cilium FAQ - Overlay 模式下 Webhook 连接超时](../networking/cilium/install.md#overlay-模式下-webhookvalidatingmutating连接超时)。

:::tip[如何判断自己是哪种模式]

`kubectl get pod -A -o wide` 查看 Pod IP：若是节点所在 VPC 网段（如 `10.10.x.x`）则为 Native 模式；若是独立的 overlay 网段（如 `10.244.x.x`）则为 Overlay 模式。三种架构的区分详见 [Cilium 概述](../networking/cilium/overview.md)。

:::

### Native 模式：替换 certgen 镜像

启用 webhook 时，certgen job 需要拉取 `kube-webhook-certgen` 镜像。新版 chart（如 87.x）该镜像默认来自 `ghcr.io/jkroepke/kube-webhook-certgen`，国内集群通常拉不到，需替换为可达的 DockerHub mirror：

```yaml title="image-values.yaml"
prometheusOperator:
  admissionWebhooks:
    patch:
      image:
        registry: docker.io
        repository: dyrnq/kube-webhook-certgen
        tag: "v1.4.4"
```

:::warning[务必先验证镜像可达]

certgen 镜像的社区 mirror 版本较多且 tag 命名不一（有的带 `v` 前缀有的不带）。选定后请务必用[前文的临时 Pod 方法](#国内环境替换镜像地址)验证节点能否拉取——DockerHub 上不存在的 tag 会导致加速器回源超时，certgen job 卡住后 operator 也会一直无法就绪。`docker.io/dyrnq/kube-webhook-certgen:v1.4.4` 经验证在 TKE 节点可拉取。

:::

### Overlay 模式：禁用 Admission Webhooks

Overlay 模式下 webhook 不可达，建议禁用。**注意：仅设置 `admissionWebhooks.enabled: false` 会导致 operator Pod 卡在 `ContainerCreating`**——因为 operator 的 TLS 开关（`prometheusOperator.tls.enabled`）默认独立为 `true`，deployment 仍会挂载 `kube-prometheus-stack-admission` Secret，而禁用 webhook 后已没有 certgen job 去生成该 Secret。正确做法是**同时关闭 operator 的 TLS**：

```yaml title="grafana-values.yaml"
prometheusOperator:
  admissionWebhooks:
    enabled: false
  # highlight-add-start
  tls:
    enabled: false
  # highlight-add-end
```

关闭 `tls.enabled` 后，operator 不再监听 HTTPS、不再挂载证书 Secret，也就不存在缺失 Secret 的问题，无需手动创建任何证书。

### cert-manager Webhook

如果集群中安装了 cert-manager，其 webhook 同样受 overlay 不可达影响。解决方案：

1. **将 cert-manager webhook 配置为 `hostNetwork: true`**（推荐）
2. **临时删除 ValidatingWebhookConfiguration**（绕过验证，适合初始部署阶段）

```bash
# 临时绕过 cert-manager webhook 验证
kubectl delete validatingwebhookconfiguration cert-manager-webhook
```

## 暴露 Grafana

### 通过 Gateway API 暴露

如果集群中已部署 EnvoyGateway，可以通过 HTTPRoute 暴露 Grafana：

```yaml title="grafana-httproute.yaml"
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
  namespace: monitoring
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: eg
    namespace: envoy-gateway-system
    sectionName: https
  hostnames:
  - "grafana.imroc.cc"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - group: ""
      kind: Service
      name: kube-prometheus-stack-grafana
      port: 80
      weight: 1
```

### 通过 port-forward 临时访问

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

访问 `http://localhost:3000`，使用 `grafana-values.yaml` 中配置的账号密码登录。

## 验证

```bash
# 检查所有 Pod 是否就绪
kubectl -n monitoring get pod

# 验证 Prometheus
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
curl http://localhost:9090/-/healthy

# 验证 Grafana
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
curl http://localhost:3000/api/health

# 获取 Grafana 密码
kubectl -n monitoring get secret kube-prometheus-stack-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d
```

## 常见问题

### Pod 一直 ImagePullBackOff？

检查 `image-values.yaml` 中的镜像配置是否正确：

1. 镜像 tag 是否与 chart 版本对应的 app 版本一致
2. 镜像仓库是否在集群节点上可达（参考[国内环境替换镜像地址](#国内环境替换镜像地址)）

### Prometheus Operator 卡在 ContainerCreating？

`kubectl describe pod` 若看到 `MountVolume.SetUp failed for volume "tls-secret" : secret "kube-prometheus-stack-admission" not found`，说明禁用了 `admissionWebhooks.enabled` 但没有同步关闭 `prometheusOperator.tls.enabled`。此时 operator 仍要挂载 admission 证书 Secret，而该 Secret 已无 certgen job 生成。

解决办法是同时关闭 operator 的 TLS，参考 [Overlay 模式：禁用 Admission Webhooks](#overlay-模式禁用-admission-webhooks)。

### certgen job 卡住导致 operator 无法就绪？

启用 webhook 时，若 `kube-prometheus-stack-admission-create` job 一直拉不到 `kube-webhook-certgen` 镜像，admission Secret 就不会生成，operator 也会一直卡在 `ContainerCreating`。参考 [Native 模式：替换 certgen 镜像](#native-模式替换-certgen-镜像) 替换为可达镜像。

### Grafana sidecar CrashLoopBackOff？

Grafana sidecar（`grafana-sc-dashboard` / `grafana-sc-datasources`）通过 Kubernetes API 列举 Secret 和 ConfigMap。在自建 Cilium Overlay 集群中，如果 sidecar Pod 无法连接 apiserver（虽然 Overlay Pod 到 apiserver 的 169.254 地址通常可达，但可能因证书验证失败而报错），可设置环境变量跳过 TLS 验证：

```yaml
grafana:
  env:
    - name: SKIP_TLS_VERIFY
      value: "true"
```
