# Kubernetes Agent Sandbox Helm Chart

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/dovakiin0-agent-sandbox)](https://artifacthub.io/packages/helm/dovakiin0-agent-sandbox/agent-sandbox)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-%3E%3D1.26.0-brightgreen.svg)](https://kubernetes.io/)

All-in-one Helm chart to deploy and manage [Kubernetes Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox) (a Kubernetes SIGs project). Provides isolated execution sandboxes for AI agents and LLM applications with a high-performance Router, pre-warmed pools, strict network security policies, and client RBAC.

---

## Features

- **Controller & Extensions**: Installs the Agent Sandbox controller with CRD support for `Sandboxes`, `SandboxClaims`, `SandboxTemplates`, and `SandboxWarmPools`.
- **Sandbox Router**: High-throughput reverse proxy for routing HTTP & WebSocket traffic directly to running sandboxes.
- **Python & Shell Runtime**: Preconfigured default `SandboxTemplate` equipped with Python 3, Bash, and the sandbox agent daemon.
- **Warm Pools**: Low-latency sandbox provisioning with configurable warm replica pools.
- **Security Hardening**:
  - `automountServiceAccountToken: false` (prevents API token exfiltration)
  - Non-root user execution (`uid 1000`) with all Linux capabilities dropped
  - Support for container isolation runtimes (`gVisor`, `Kata Containers`)
  - Strict `NetworkPolicy` blocking RFC1918 private subnets and cloud metadata endpoints (`169.254.169.254`).
- **Client RBAC**: Pre-configured `ClusterRole` and `ClusterRoleBinding` for backend services and AI agent orchestrators.

---

## Prerequisites

- **Kubernetes**: `>= 1.26.0`
- **Helm**: `>= 3.12.0`
- *(Optional)* Container isolation runtime such as **gVisor** (`runsc`) or **Kata Containers** configured on worker nodes.

---

## Installation

### 1. Add the Helm Repository

```bash
helm repo add dovakiin0 https://dovakiin0.github.io/agent-sandbox-helm/
helm repo update
```

### 2. Install the Chart

Install into a dedicated namespace (e.g. `agent-sandbox-system`):

```bash
helm install agent-sandbox dovakiin0/agent-sandbox \
  --namespace agent-sandbox-system \
  --create-namespace
```

---

## Configuration Reference

The following table lists the configurable parameters of the chart and their default values:

### Controller Settings

| Parameter | Description | Default |
|-----------|-------------|---------|
| `controller.replicaCount` | Number of controller replicas | `1` |
| `controller.image.repository` | Controller image repository | `registry.k8s.io/agent-sandbox/agent-sandbox-controller` |
| `controller.image.tag` | Controller image tag | `v1.0.2` |
| `controller.image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `controller.leaderElect` | Enable leader election | `"true"` |
| `controller.extensions.enabled` | Enable extensions controller (warmpools, templates) | `true` |
| `controller.resources.requests.cpu` | CPU request | `100m` |
| `controller.resources.requests.memory` | Memory request | `128Mi` |
| `controller.resources.limits.cpu` | CPU limit | `500m` |
| `controller.resources.limits.memory` | Memory limit | `512Mi` |

### Router Settings

| Parameter | Description | Default |
|-----------|-------------|---------|
| `router.replicaCount` | Number of router replicas | `2` |
| `router.image.repository` | Router image repository | `us-central1-docker.pkg.dev/k8s-staging-images/agent-sandbox/sandbox-router` |
| `router.image.tag` | Router image tag | `latest-main` |
| `router.service.type` | Router service type | `ClusterIP` |
| `router.service.port` | Router service port | `8080` |
| `router.proxyTimeoutSeconds` | HTTP proxy request timeout (seconds) | `180` |
| `router.websocketIdleTimeoutSeconds` | WebSocket idle timeout (seconds) | `3600` |
| `router.websocketMaxLifetimeSeconds` | WebSocket maximum lifetime (seconds) | `86400` |
| `router.websocketMaxMessageBytes` | WebSocket max message payload size | `16777216` |
| `router.allowUnauthenticated` | Allow unauthenticated proxying | `"true"` |
| `router.topologySpreadConstraints.enabled` | Distribute across failure zones | `true` |
| `router.resources.requests.cpu` | CPU request | `100m` |
| `router.resources.requests.memory` | Memory request | `256Mi` |
| `router.resources.limits.cpu` | CPU limit | `1000m` |
| `router.resources.limits.memory` | Memory limit | `1Gi` |

### Sandbox Template Settings

| Parameter | Description | Default |
|-----------|-------------|---------|
| `sandboxTemplate.name` | Name of the default SandboxTemplate | `python-bash-template` |
| `sandboxTemplate.image.repository` | Runtime image repository | `us-central1-docker.pkg.dev/k8s-staging-images/agent-sandbox/python-runtime-sandbox` |
| `sandboxTemplate.image.tag` | Runtime image tag | `latest-main` |
| `sandboxTemplate.resources.requests.cpu` | Sandbox CPU request | `250m` |
| `sandboxTemplate.resources.requests.memory` | Sandbox memory request | `512Mi` |
| `sandboxTemplate.resources.limits.cpu` | Sandbox CPU limit | `1000m` |
| `sandboxTemplate.resources.limits.memory` | Sandbox memory limit | `1Gi` |
| `sandboxTemplate.persistence.enabled` | Attach persistent volume claim template | `false` |
| `sandboxTemplate.persistence.size` | PVC request size | `1Gi` |

### Security Hardening

| Parameter | Description | Default |
|-----------|-------------|---------|
| `security.runtimeClassName` | Container isolation runtime (e.g. `gvisor`, `kata-qemu`) | `""` |
| `security.runAsUser` | Sandbox container UID | `1000` |
| `security.runAsGroup` | Sandbox container GID | `1000` |
| `security.readOnlyRootFilesystem` | Mount root filesystem as read-only | `false` |

### Warm Pools

| Parameter | Description | Default |
|-----------|-------------|---------|
| `warmPool.name` | Name of the default SandboxWarmPool | `python-bash-warmpool` |
| `warmPool.replicas` | Pre-warmed standby sandbox count | `2` |
| `warmPool.updateStrategy` | Warm pool update strategy | `OnReplenish` |

### Network Policy & Backend RBAC

| Parameter | Description | Default |
|-----------|-------------|---------|
| `networkPolicy.enabled` | Enforce strict NetworkPolicy on sandboxes | `true` |
| `networkPolicy.egress.allowPublicInternet` | Allow egress to internet (blocking metadata & RFC1918) | `true` |
| `networkPolicy.egress.additionalEgress` | Custom additional egress rules | `[]` |
| `backendRBAC.enabled` | Deploy RBAC for backend service managing sandboxes | `true` |
| `backendRBAC.serviceAccount.name` | Backend ServiceAccount name | `agent-sandbox-backend` |
| `backendRBAC.serviceAccount.namespace` | Backend ServiceAccount namespace | `claros-sandbox` |
| `backendRBAC.additionalSubjects` | Additional service accounts to grant permissions | `[]` |

---

## Customizing Values

To customize values during installation:

```bash
cat <<EOF > custom-values.yaml
security:
  runtimeClassName: "gvisor"

warmPool:
  replicas: 5

sandboxTemplate:
  resources:
    limits:
      cpu: "2000m"
      memory: "2Gi"
EOF

helm install agent-sandbox dovakiin0/agent-sandbox \
  -f custom-values.yaml \
  --namespace agent-sandbox-system
```

---

## Upstream Documentation & Resources

- [Official Kubernetes Agent Sandbox Docs](https://agent-sandbox.sigs.k8s.io)
- [GitHub: kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox)
- [Chart Source Repository](https://github.com/Dovakiin0/agent-sandbox-helm)
