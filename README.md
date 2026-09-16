# Agent Sandbox Helm Charts

[![Release Charts](https://github.com/Dovakiin0/agent-sandbox-helm/actions/workflows/release.yaml/badge.svg)](https://github.com/Dovakiin0/agent-sandbox-helm/actions/workflows/release.yaml)
[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/dovakiin0)](https://artifacthub.io/packages/search?repo=dovakiin0)
[![Latest Release](https://img.shields.io/github/v/release/Dovakiin0/agent-sandbox-helm)](https://github.com/Dovakiin0/agent-sandbox-helm/releases)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Helm repository for **[Kubernetes Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox)** (`sigs.k8s.io/agent-sandbox`).

This repository packages and maintains Helm charts to deploy and manage isolated execution sandboxes for AI agents, LLMs, and code execution workloads on Kubernetes.

---

## Charts Included

| Chart | Version | Description | Artifact Hub |
| --- | --- | --- | --- |
| [`agent-sandbox`](charts/agent-sandbox) | [![Latest Release](https://img.shields.io/github/v/release/Dovakiin0/agent-sandbox-helm?label=version)](https://github.com/Dovakiin0/agent-sandbox-helm/releases) | All-in-one chart bundling Controller, Router, Python/Bash runtime, WarmPools, NetworkPolicy, and RBAC | [View on Artifact Hub](https://artifacthub.io/packages/helm/dovakiin0-agent-sandbox/agent-sandbox) |

---

## Quick Start

### 1. Add the Helm Repository

```bash
helm repo add dovakiin0 https://dovakiin0.github.io/agent-sandbox-helm/
helm repo update
```

### 2. Install the Agent Sandbox Chart

```bash
helm install agent-sandbox dovakiin0/agent-sandbox \
  --namespace agent-sandbox-system \
  --create-namespace
```

### 3. Verify Deployment

Check that the Controller, Router, and WarmPool are running:

```bash
kubectl get pods -n agent-sandbox-system
kubectl get sandboxtemplates,sandboxwarmpools,sandboxes -n agent-sandbox-system
```

---

## Architecture and Key Components

```
                +------------------------------------+
                |       Client / AI Orchestrator     |
                +-----------------+------------------+
                                  |
            +---------------------+--------------------+
            | HTTP / WebSocket                         | Sandbox CRD / Claim
            v                                          v
+-----------------------+                  +-----------------------+
|    Sandbox Router     |                  | Agent Sandbox Ctrl    |
| (High-throughput      |                  | (Reconciles Sandbox,  |
|  Reverse Proxy)       |                  |  WarmPools, Templates)|
+-----------+-----------+                  +-----------+-----------+
            |                                          |
            |                                          |
            v                                          v
+------------------------------------------------------------------+
|                   Isolated Sandbox Pods                          |
|  - automountServiceAccountToken: false                           |
|  - Non-root (UID 1000) / Dropped Capabilities                    |
|  - gVisor / Kata Isolation Support                               |
|  - Strict NetworkPolicy (Blocks 169.254.169.254 & RFC1918)      |
+------------------------------------------------------------------+
```

1. **Agent Sandbox Controller**: Core controller with extension support managing lifecycle of Sandboxes, SandboxClaims, and WarmPools.
2. **Sandbox Router**: Scales independently with zone topology spread, forwarding incoming traffic to targeted sandbox instances over HTTP/WebSocket.
3. **Sandbox Templates & Warm Pools**: Instantly satisfies sandbox claims with zero cold-start delay from a warm standby pool.
4. **Security Hardening**: Built-in NetworkPolicy isolating sandboxes from internal cluster networks and cloud metadata servers, and gVisor/Kata isolation support.

---

## Configuration and Customization

See the detailed configuration guide and parameters list in the [Chart README](charts/agent-sandbox/README.md).

Example custom configuration (`values.yaml`):

```yaml
# Enable gVisor VM/kernel isolation
security:
  runtimeClassName: "gvisor"

# Scale pre-warmed pool
warmPool:
  replicas: 5

# Customize sandbox resources
sandboxTemplate:
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2000m"
      memory: "2Gi"
```

Apply customization:

```bash
helm upgrade --install agent-sandbox dovakiin0/agent-sandbox \
  -f values.yaml \
  --namespace agent-sandbox-system
```

---

## Repository Structure

```
agent-sandbox-helm/
├── .github/
│   └── workflows/
│       └── release.yaml              # Automated chart release & GitHub Pages workflow
├── charts/
│   └── agent-sandbox/                # Helm chart
│       ├── Chart.yaml                # Chart metadata and Artifact Hub annotations
│       ├── README.md                 # Artifact Hub package README
│       ├── values.yaml               # Default values
│       ├── values.schema.json        # Values JSON schema
│       ├── crds/                     # Agent Sandbox Custom Resource Definitions
│       │   ├── sandboxes.agents.x-k8s.io.yaml
│       │   ├── sandboxclaims.extensions.agents.x-k8s.io.yaml
│       │   ├── sandboxtemplates.extensions.agents.x-k8s.io.yaml
│       │   └── sandboxwarmpools.extensions.agents.x-k8s.io.yaml
│       └── templates/                # Kubernetes resource templates
│           ├── _helpers.tpl
│           ├── controller-deployment.yaml
│           ├── controller-rbac.yaml
│           ├── router.yaml
│           ├── sandbox-template.yaml
│           ├── sandbox-warmpool.yaml
│           ├── networkpolicy.yaml
│           └── backend-rbac.yaml
├── artifacthub-repo.yml              # Artifact Hub repository verification
└── README.md
```

---

## Publishing and Releases

This repository uses [Helm Chart Releaser](https://github.com/helm/chart-releaser-action) to automatically publish new chart versions:

1. Bump `version` in `charts/agent-sandbox/Chart.yaml`.
2. Commit and push to `main` or `master`:
   ```bash
   git add .
   git commit -m "Release agent-sandbox v1.0.3"
   git push origin master
   ```
3. GitHub Actions builds the chart package, creates a GitHub Release, and publishes the Helm index to the `gh-pages` branch.
4. Artifact Hub automatically detects and lists the updated chart.

---

## License

This project is licensed under the [Apache 2.0 License](LICENSE).

Upstream project: [Kubernetes SIGs Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox).
