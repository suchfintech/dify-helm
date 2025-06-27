# Deploying Dify with Flux on MicroK8s

This guide demonstrates deploying the Dify Helm chart in a MicroK8s cluster managed by Flux. The example assumes that PostgreSQL and Redis are already available in the cluster and that `cert-manager` is installed for TLS certificates.

## Prerequisites

1. **MicroK8s** with addons `helm3`, `dns`, and `ingress` enabled.
2. **FluxCD** installed in the cluster and bootstrapped against your Git repository.
3. Existing **PostgreSQL** and **Redis** services reachable within the cluster.
4. **cert-manager** with a configured ClusterIssuer (for example `letsencrypt`).

## Add the Helm Repository

Flux consumes charts from a `HelmRepository` resource. Add the public Dify repository to your Git sources:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: dify
  namespace: flux-system
spec:
  url: https://borispolonsky.github.io/dify-helm
  interval: 1h
```

Apply it to the cluster via Git commit so Flux can reconcile it.

## HelmRelease Example

Create a `HelmRelease` to install the chart. The values below use existing PostgreSQL and Redis instances and enable an ingress secured by cert-manager.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: dify
  namespace: dify
spec:
  interval: 5m
  chart:
    spec:
      chart: dify
      version: "0.x"
      sourceRef:
        kind: HelmRepository
        name: dify
        namespace: flux-system
  valuesFrom:
    - kind: ConfigMap
      name: dify-values
```
```
Create a `ConfigMap` named `dify-values` with your custom configuration:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dify-values
  namespace: dify
```
Add a `values-production.yaml` to this ConfigMap as `data.values.yaml`. For example:
```yaml
postgresql:
  enabled: false
redis:
  enabled: false
externalPostgres:
  enabled: true
  address: postgres-cluster.default.svc.cluster.local
  port: 5432
  username: dify
  password: "changeme"
  database:
    api: dify
    pluginDaemon: dify_plugin
externalRedis:
  enabled: true
  host: redis-master.default.svc.cluster.local
  port: 6379
  password: "changeme"
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
  hosts:
    - host: dify.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: dify-tls
      hosts:
        - dify.example.com
```
Commit the `HelmRepository`, `HelmRelease`, and ConfigMap manifests to the repository watched by Flux. Flux will then deploy Dify with the provided values.

## Values Overview

The chart's `values.yaml` is organised into several sections:

- **Image settings** – override image repositories and tags for each component.
- **Component options** – replicas, resources, probes, and `extraEnv` for `api`, `worker`, `web`, `proxy`, `sandbox`, and `pluginDaemon`.
- **Built-in middleware** – optional PostgreSQL, Redis, and Weaviate subcharts.
- **External services** – configure `externalPostgres`, `externalRedis`, object storage providers (`externalS3`, `externalAzureBlobStorage`, etc.), and vector databases (`externalQdrant`, `externalMilvus`, `externalPgvector`, ...).
- **Ingress** – enable external access and annotate for cert-manager TLS.

Refer to the comments in `values.yaml` for the complete list of options and defaults.

## Environment Variables and Secrets

`config.tpl` and `credentials.tpl` generate ConfigMaps and Secrets used by all pods. Database credentials, Redis passwords, API keys, and storage credentials are populated from the chart values. Additional variables can be set via the `extraEnv` fields per component.

---
For a more complete production example, see `values-production.yaml` in the chart root.

