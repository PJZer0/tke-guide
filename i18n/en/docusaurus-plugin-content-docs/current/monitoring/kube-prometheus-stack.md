---
sidebar_position: 1
---

# Building Monitoring System Using kube-prometheus-stack

## Overview

[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) is a helm chart in the Prometheus ecosystem for deploying Prometheus-related components in Kubernetes, covering Prometheus Operator, Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, and various Grafana dashboards provided by the community. This article describes how to use this chart to build a monitoring system in a TKE cluster.

## Installation

Add the helm repo:

```bash
helm repo add prom https://prometheus-community.github.io/helm-charts
helm repo update
```

Install:

```bash
helm upgrade --install kube-prometheus-stack prom/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --version <chart-version> \
  -f image-values.yaml \
  -f grafana-values.yaml
```

:::tip[Choose the Right Chart Version]

The chart version of `kube-prometheus-stack` corresponds one-to-one with the app version (Prometheus Operator version). CRDs, default configurations, and image versions may differ significantly between versions.

After selecting a chart version, the image tags in `image-values.yaml` must match the app version corresponding to the chart version, otherwise compatibility issues may arise.

:::

## Custom Configuration Methods

The `kube-prometheus-stack` chart is very large with numerous configuration options. It is recommended to split custom configurations into multiple `values.yaml` files for separate maintenance, and specify multiple `-f` parameters during installation:

- `image-values.yaml`: Image replacement configuration
- `grafana-values.yaml`: Grafana and other custom configurations

```bash
helm upgrade --install kube-prometheus-stack prom/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f image-values.yaml \
  -f grafana-values.yaml
```

If using kustomize for management, you can use `additionalValuesFiles`:

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

## Replacing Image Addresses for Domestic Environments

The images used by `kube-prometheus-stack` primarily come from `quay.io`, which may fail or timeout when pulling domestically. There are two solutions:

### Using TKE Internal Mirror (Recommended)

TKE provides `quay.tencentcloudcr.com` as an internal mirror for `quay.io`. Simply replace the image registry:

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

Some images not on `quay.io` can be replaced with community mirrors on DockerHub:

| Original Image                                        | DockerHub Mirror Image                 |
| :---------------------------------------------------- | :------------------------------------- |
| registry.k8s.io/kube-state-metrics/kube-state-metrics | docker.io/k8smirror/kube-state-metrics |

:::tip[Verify Image Reachability Within the Cluster]

Different clusters configure image acceleration differently on their nodes (e.g., TKE nodes typically point `docker.io` to the internal accelerator `mirror.ccs.tencentyun.com`). After choosing a mirror image, it's recommended to verify that nodes can pull it using a temporary Pod, to avoid discovering unreachable images only at install time:

```bash
kubectl run img-test --image=docker.io/xxx/yyy:tag --restart=Never --command -- true
kubectl describe pod img-test | grep -A5 Events   # "Successfully pulled" means reachable
kubectl delete pod img-test
```

As long as the image **actually exists on DockerHub**, it can be pulled via the internal accelerator. If the image/tag does not exist on DockerHub, the accelerator's fallback to the origin will time out (`i/o timeout`) rather than fail fast, which is easy to misdiagnose.

:::

## Configuring Grafana

Grafana is a subchart of `kube-prometheus-stack`. All Grafana configurations are placed under the `grafana` field:

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

For specific configuration recommendations, refer to [Self-hosting Grafana on TKE](./grafana).

## Admission Webhooks Configuration

The Prometheus Operator enables admission webhooks by default, used to validate the syntax of CRDs such as `PrometheusRule` (e.g., catching invalid alert rule expressions early). The webhook's TLS certificate is automatically generated by the chart's built-in certgen job into the `kube-prometheus-stack-admission` Secret, which the operator mounts and uses.

**Whether you need to disable the webhook depends on the cluster's network mode:**

- **Cilium Native (VPC-CNI) mode**: Pod IPs are real VPC IPs, and the apiserver can route directly to Pods. The webhook works normally and **does not need to be disabled** — you only need to handle the certgen image (see below).
- **Cilium Overlay mode**: The apiserver runs on the control plane (without cilium-agent) and cannot route to overlay Pod IPs (e.g., `10.244.x.x`), so apiserver calls to the webhook will time out. It is **recommended to disable it** (see below). See [Installing Cilium FAQ - Webhook Validating/Mutating Connection Timeout in Overlay Mode](../networking/cilium/install.md#webhook-validatingmutating-connection-timeout-in-overlay-mode).

:::tip[How to Determine Your Mode]

Run `kubectl get pod -A -o wide` to check Pod IPs: if they are in the node's VPC CIDR (e.g., `10.10.x.x`), it's Native mode; if they are in a separate overlay CIDR (e.g., `10.244.x.x`), it's Overlay mode. For the distinction between the three architectures, see [Cilium Overview](../networking/cilium/overview.md).

:::

### Native Mode: Replacing the certgen Image

When the webhook is enabled, the certgen job needs to pull the `kube-webhook-certgen` image. In newer chart versions (e.g., 87.x), this image defaults to `ghcr.io/jkroepke/kube-webhook-certgen`, and domestic clusters usually cannot pull from `ghcr.io`. Fortunately, the image maintainer [jkroepke](https://github.com/jkroepke/kube-webhook-certgen) also publishes to DockerHub, so simply changing the registry to `docker.io` works (the tag matches the chart default, no change needed):

```yaml title="image-values.yaml"
prometheusOperator:
  admissionWebhooks:
    patch:
      image:
        registry: docker.io
        repository: jkroepke/kube-webhook-certgen
        tag: "1.8.4"
```

:::tip[Prefer the Official Image, and Verify Reachability]

`docker.io/jkroepke/kube-webhook-certgen` is the official DockerHub publication by the same author as the chart's default image, which is more reliable than third-party community mirrors. Its tags match the `ghcr.io` versions (in this example `1.8.4` is the chart 87.3.0 default). After replacing, it's still recommended to verify that nodes can pull it using the [temporary Pod method above](#replacing-image-addresses-for-domestic-environments) — a non-existent tag on DockerHub will cause the accelerator's origin fallback to time out, leaving the certgen job stuck and the operator perpetually not ready. `docker.io/jkroepke/kube-webhook-certgen:1.8.4` has been verified to be pullable on TKE nodes.

:::

### Overlay Mode: Disabling Admission Webhooks

In Overlay mode the webhook is unreachable, so it's recommended to disable it. **Note: setting only `admissionWebhooks.enabled: false` will leave the operator Pod stuck in `ContainerCreating`** — because the operator's TLS switch (`prometheusOperator.tls.enabled`) is independently `true` by default, the deployment still mounts the `kube-prometheus-stack-admission` Secret, but once the webhook is disabled there is no longer a certgen job to generate that Secret. The correct approach is to **also disable the operator's TLS**:

```yaml title="grafana-values.yaml"
prometheusOperator:
  admissionWebhooks:
    enabled: false
  # highlight-add-start
  tls:
    enabled: false
  # highlight-add-end
```

After disabling `tls.enabled`, the operator no longer listens on HTTPS or mounts the certificate Secret, so the missing-Secret problem disappears and no manual certificate creation is needed.

### cert-manager Webhook

If cert-manager is installed in the cluster, its webhook is also affected by the overlay unreachability. Solutions:

1. **Configure the cert-manager webhook with `hostNetwork: true`** (recommended)
2. **Temporarily delete the ValidatingWebhookConfiguration** (bypasses validation, suitable for initial deployment phase)

```bash
# Temporarily bypass cert-manager webhook validation
kubectl delete validatingwebhookconfiguration cert-manager-webhook
```

## Exposing Grafana

### Exposing via Gateway API

If EnvoyGateway is already deployed in the cluster, you can expose Grafana through an HTTPRoute:

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

### Temporary Access via port-forward

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Access `http://localhost:3000` and log in with the credentials configured in `grafana-values.yaml`.

## Verification

```bash
# Check if all Pods are ready
kubectl -n monitoring get pod

# Verify Prometheus
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
curl http://localhost:9090/-/healthy

# Verify Grafana
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
curl http://localhost:3000/api/health

# Get Grafana password
kubectl -n monitoring get secret kube-prometheus-stack-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d
```

## FAQ

### Pod Stuck in ImagePullBackOff?

Check if the image configuration in `image-values.yaml` is correct:

1. Whether the image tags match the app version corresponding to the chart version
2. Whether the image registry is reachable from the cluster nodes (refer to [Replacing Image Addresses for Domestic Environments](#replacing-image-addresses-for-domestic-environments))

### Prometheus Operator Stuck in ContainerCreating?

If `kubectl describe pod` shows `MountVolume.SetUp failed for volume "tls-secret" : secret "kube-prometheus-stack-admission" not found`, it means `admissionWebhooks.enabled` was disabled without also disabling `prometheusOperator.tls.enabled`. The operator still tries to mount the admission certificate Secret, but no certgen job generates it anymore.

The fix is to also disable the operator's TLS. See [Overlay Mode: Disabling Admission Webhooks](#overlay-mode-disabling-admission-webhooks).

### certgen Job Stuck, Operator Not Ready?

When the webhook is enabled, if the `kube-prometheus-stack-admission-create` job keeps failing to pull the `kube-webhook-certgen` image, the admission Secret won't be generated and the operator will also stay stuck in `ContainerCreating`. See [Native Mode: Replacing the certgen Image](#native-mode-replacing-the-certgen-image) to replace it with a reachable image.

### Grafana sidecar CrashLoopBackOff?

Grafana sidecars (`grafana-sc-dashboard` / `grafana-sc-datasources`) list Secrets and ConfigMaps via the Kubernetes API. In self-managed Cilium Overlay clusters, if the sidecar Pod cannot connect to the apiserver (although Overlay Pods can usually reach the apiserver at the 169.254 address, certificate verification failures may cause errors), you can set an environment variable to skip TLS verification:

```yaml
grafana:
  env:
    - name: SKIP_TLS_VERIFY
      value: "true"
```
