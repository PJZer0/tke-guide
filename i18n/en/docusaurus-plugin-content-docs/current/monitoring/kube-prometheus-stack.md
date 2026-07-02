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

## Updating CRDs When Upgrading Across Versions

:::warning[CRDs Must Be Updated First for Major Version Upgrades]

Each major version of `kube-prometheus-stack` (e.g., 80→81, 86→87) is usually accompanied by a Prometheus Operator upgrade, and the CRDs are also updated. Helm does not update installed CRDs by default, so this must be handled manually — otherwise the Operator may fail to recognize new fields after the upgrade.

:::

The chart provides the `crds.upgradeJob` option (v68.4.0+). When enabled, it automatically updates CRDs via a Helm hook during installation/upgrade:

```yaml title="image-values.yaml"
crds:
  upgradeJob:
    enabled: true
    # If CRDs are already managed via SSA by GitOps tools like ArgoCD, conflicting fields must be force-overwritten
    forceConflicts: true
    image:
      kubectl:
        # kubectl is pulled from registry.k8s.io by default; replace with a mirror for domestic environments
        # The tag defaults to "v" + the K8s version (e.g., v1.34.1); inheriting the chart default is fine
        registry: docker.io
        repository: k8smirror/kubectl
```

:::tip[When to Use forceConflicts]

If the cluster is managed by GitOps tools like ArgoCD, the CRDs are taken over by ArgoCD's SSA (Server-Side Apply). In this case, the upgradeJob's `kubectl apply --server-side` will conflict with the `argocd-controller` on certain fields. Setting `forceConflicts: true` lets kubectl force-overwrite the conflicting fields to complete the CRD upgrade.

:::

If you prefer not to use upgradeJob, you can also manually apply the latest CRDs:

```bash
# Replace <version> with the target Prometheus Operator version
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/<version>/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml
# Apply the remaining CRDs similarly
```

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
  version: "87.5.1" # replace with the latest version as needed
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

For the vast majority of clusters (ordinary TKE clusters), the webhook works out of the box — you only need to handle the certgen job's image address.

### Replacing the certgen Image

The certgen job needs to pull the `kube-webhook-certgen` image, which the chart defaults to `ghcr.io/jkroepke/kube-webhook-certgen`, and domestic clusters usually cannot pull from `ghcr.io`. Fortunately, the image maintainer [jkroepke](https://github.com/jkroepke/kube-webhook-certgen) also publishes to DockerHub, so simply changing the registry to `docker.io` works (both repository and tag match the chart default, no override needed):

```yaml title="image-values.yaml"
prometheusOperator:
  admissionWebhooks:
    patch:
      image:
        registry: docker.io
```

### Overlay Clusters: Disabling Admission Webhooks

:::warning[Only Needed for Overlay-Networked Clusters]

"Overlay" here refers to scenarios where **the apiserver cannot route directly to Pod IPs**, typically: **self-managed Cilium Overlay** or **productized Cilium Overlay** managed clusters — the apiserver runs on the control plane and cannot route to overlay Pod IPs (e.g., `10.244.x.x`), so calls to the webhook time out.

Ordinary TKE clusters (where Pod IPs are real VPC IPs and the apiserver can reach Pods directly) **do not need** the configuration in this section — just enable the webhook normally as above.

:::

In Overlay clusters the webhook is unreachable, so it's recommended to disable it. **Note: setting only `admissionWebhooks.enabled: false` will leave the operator Pod stuck in `ContainerCreating`** — because the operator's TLS switch (`prometheusOperator.tls.enabled`) is independently `true` by default, the deployment still mounts the `kube-prometheus-stack-admission` Secret, but once the webhook is disabled there is no longer a certgen job to generate that Secret. The correct approach is to **also disable the operator's TLS**:

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

Additionally, if cert-manager is installed in an Overlay cluster, its webhook is also affected by the overlay unreachability. Solutions:

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

The fix is to also disable the operator's TLS. See [Overlay Clusters: Disabling Admission Webhooks](#overlay-clusters-disabling-admission-webhooks).

### certgen Job Stuck, Operator Not Ready?

When the webhook is enabled, if the `kube-prometheus-stack-admission-create` job keeps failing to pull the `kube-webhook-certgen` image, the admission Secret won't be generated and the operator will also stay stuck in `ContainerCreating`. See [Replacing the certgen Image](#replacing-the-certgen-image) to replace it with a reachable image.

### Grafana sidecar CrashLoopBackOff?

Grafana sidecars (`grafana-sc-dashboard` / `grafana-sc-datasources`) list Secrets and ConfigMaps via the Kubernetes API. In self-managed Cilium Overlay clusters, if the sidecar Pod cannot connect to the apiserver (although Overlay Pods can usually reach the apiserver at the 169.254 address, certificate verification failures may cause errors), you can set an environment variable to skip TLS verification:

```yaml
grafana:
  env:
    - name: SKIP_TLS_VERIFY
      value: "true"
```
