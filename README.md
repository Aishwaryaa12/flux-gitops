# Cloud-Native Homelab Platform

> A GitOps-managed Kubernetes platform where infrastructure, applications, and policies are defined in Git and continuously reconciled into the cluster.

---
## Table of Contents

- [Executive Summary](#executive-summary)
- [Architecture Overview](#architecture-overview)
- [Component Summary](#component-summary)
- [Repository Structure](#repository-structure)
- [GitOps Workflow](#gitops-workflow)
- [Image Automation](#image-automation)
- [Application Delivery](#application-delivery)
- [Networking Architecture](#networking-architecture)
- [Observability Stack](#observability-stack)
- [Security Architecture](#security-architecture)
- [Engineering Decisions and Tradeoffs](#engineering-decisions-and-tradeoffs)
- [Known Gaps and Honest Assessment](#known-gaps-and-honest-assessment)

---
## Executive Summary

This repository is the single source of truth for the Cloud-Native Kubernetes platform. It runs on a single-node k3s cluster on Debian 13, managed entirely by FluxCD. No resource is applied manually, namespaces, certificates, deployments, routing rules, secrets, and admission policies are all reconciled from this repository.

The platform hosts two self-hosted workloads: **Vaultwarden** at `vault.cralyx.com` and **Linkding** at `linkding.cralyx.com`. Both are served over HTTPS via Traefik's Gateway API implementation, TLS-terminated with a wildcard Let's Encrypt certificate issued through Cloudflare DNS-01 challenge.

Beyond basic application delivery, the platform runs a **closed-loop image update pipeline**: Flux scans Docker Hub hourly, evaluates new tags against semver range policies, and commits updated image references back to `main`  triggering its own reconciliation. Application updates are deployed without human intervention within the bounds of the defined policy.

Secrets are encrypted with SOPS/Age and committed to the repository. Admission policies are enforced by Kyverno. Metrics and logs flow through kube-prometheus-stack, Loki, and Grafana Alloy into a unified Grafana dashboard. The result is a platform where state is always observable, always in Git, and always reproducible from a fresh OS install.

---
## Architecture Overview


```mermaid
flowchart LR
    %% --- Styling Classes ---
    classDef gitops fill:#3b82f6,stroke:#1d4ed8,stroke-width:2px,color:#fff
    classDef infra fill:#8b5cf6,stroke:#6d28d9,stroke-width:2px,color:#fff
    classDef telemetry fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#fff
    classDef app fill:#10b981,stroke:#047857,stroke-width:2px,color:#fff
    classDef secret fill:#ef4444,stroke:#b91c1c,stroke-width:2px,color:#fff
    
    %% Brand Colors for External Services
    classDef github fill:#1e293b,stroke:#0f172a,stroke-width:2px,color:#fff
    classDef docker fill:#0ea5e9,stroke:#0369a1,stroke-width:2px,color:#fff
    classDef cloudflare fill:#f97316,stroke:#c2410c,stroke-width:2px,color:#fff

    %% --- External Dependencies ---
    subgraph External["External Resources"]
        direction TB
        REPO[("GitHub\n(Single Source of Truth)")]:::github
        DH["Docker Hub"]:::docker
        CF_API["Cloudflare API\n(ACME DNS-01)"]:::cloudflare
        CF_DNS["Cloudflare DNS\n(*.cralyx.com)"]:::cloudflare
    end

    %% --- Kubernetes Environment ---
    subgraph Cluster["Kubernetes Cluster"]
        direction TB
        
        subgraph Flux["GitOps Control Plane"]
            SRC["Source Controller"]:::gitops
            KUST["Kustomize Controller"]:::gitops
            HELM["Helm Controller"]:::gitops
            IMG["Image Reflector & Automation"]:::gitops
        end
        
        subgraph Security["Security & Secrets"]
            SOPS["SOPS / Age"]:::secret
            CF_SEC["API Token Secret"]:::secret
            CERTMGR["cert-manager"]:::infra
            WILDCARD["TLS Wildcard"]:::infra
            KYVERNO["Kyverno Policies"]:::infra
        end
        
        subgraph Network["Ingress & Gateway"]
            TRAEFIK["Traefik v3\n(Gateway API mode)"]:::infra
            GW["Gateway\n(:8443)"]:::infra
        end

        subgraph Observability["Telemetry"]
            ALLOY["Grafana Alloy"]:::telemetry
            PROM["Prometheus"]:::telemetry
            LOKI["Loki"]:::telemetry
            GRAF["Grafana"]:::telemetry
        end

        subgraph Apps["Workloads"]
            VW["vault.cralyx.com"]:::app
            LD["linkding.cralyx.com"]:::app
        end
    end

    %% --- State Reconciliation Flow ---
    REPO -->|"polls 10m"| SRC
    SRC --> KUST
    IMG -.->|"scans 1h"| DH
    IMG -->|"commits tag"| REPO

    %% --- Kustomize Layering ---
    KUST -->|"Layer 1"| SOPS
    KUST -->|"Layer 2"| Network
    KUST -->|"Layer 3"| Apps
    
    %% --- Helm Management ---
    HELM -.->|"manages"| Observability
    HELM -.->|"manages"| Security

    %% --- Secrets & Certificate Flow ---
    SOPS --> CF_SEC --> CERTMGR
    CERTMGR <-->|"DNS-01 auth"| CF_API
    CERTMGR --> WILDCARD --> GW

    %% --- External Traffic Flow ---
    CF_DNS -->|"HTTPS :443 → :8443"| GW
    GW -->|"HTTPRoute"| VW & LD

    %% --- Telemetry Flow ---
    ALLOY -->|"metrics"| PROM
    ALLOY -->|"logs"| LOKI
    PROM & LOKI --> GRAF
```


---

## Component Summary

| Component                 | Role                    | Why This Choice                                                                                             |
| ------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Debian 13**             | Host OS                 | Stability, long support lifecycle, minimal overhead                                                         |
| **k3s**                   | Kubernetes distribution | Single-binary, production-capable, ships Traefik and CoreDNS; minimal operational overhead                  |
| **FluxCD**                | GitOps engine           | Pull-based reconciliation; image automation built-in; no external CI pipeline required                      |
| **Traefik v3**            | Ingress + Gateway API   | Bundled with k3s; Gateway API support enabled via `HelmChartConfig`; avoids a second ingress controller     |
| **Gateway API**           | Traffic routing         | Supersedes Ingress; clean separation between infrastructure (Gateway) and application (HTTPRoute) ownership |
| **Cloudflare**            | DNS                     | DNS-01 ACME challenge support; wildcard cert issuance without exposing HTTP endpoints publicly              |
| **cert-manager**          | TLS lifecycle           | De facto Kubernetes standard; automatic ACME + Cloudflare integration; handles renewal                      |
| **kube-prometheus-stack** | Metrics + dashboards    | Bundles Prometheus, Grafana, node-exporter, and recording rules in one release                              |
| **Loki**                  | Log aggregation         | Label-indexed; pairs naturally with Kubernetes metadata; native Grafana integration                         |
| **Grafana Alloy**         | Telemetry pipeline      | Unified replacement for Promtail and Grafana Agent; single DaemonSet for logs and metrics                   |
| **Kyverno**               | Admission policies      | Native Kubernetes YAML policies; no Rego required; supports background scanning of existing resources       |
| **SOPS + Age**            | Secret encryption       | Asymmetric encryption; Git-compatible diffs; no external secret store dependency                            |
| **Vaultwarden**           | Password manager        | Self-hosted Bitwarden; critical personal infrastructure                                                     |
| **Linkding**              | Bookmark manager        | Lightweight, minimal resource footprint                                                                     |

---

## Repository Structure


```
.
├── apps/                               # Application workloads — owned by app namespaces
│   ├── kustomization.yaml
│   ├── linkding/
│   │   ├── deployment.yaml             # Namespace, Deployment (pinned image + resource limits), Service
│   │   ├── certificate.yaml
│   │   ├── pvc.yaml                    # PVC → /etc/linkding/data
│   │   └── kustomization.yaml
│   └── vaultwarden/
│       ├── deployment.yaml             # Namespace, Deployment, Service; SIGNUPS_ALLOWED=false
│       ├── certificate.yaml
│       ├── pvc.yaml                    # PVC → /data
│       └── kustomization.yaml
│
├── clusters/
│   └── production/                     # ← single cluster entrypoint
│       ├── kustomization.yaml          # Root composer — references all layer Kustomization CRs
│       ├── flux-system/                # Flux bootstrap manifests (gotk-components, gotk-sync)
│       ├── secrets.yaml                # Kustomization CR: ./secrets · SOPS decrypt enabled
│       ├── infrastructure.yaml         # Kustomization CR: ./infrastructure · dependsOn: secrets
│       ├── policies.yaml               # Kustomization CR: ./policies/kyverno · dependsOn: infrastructure
│       └── apps.yaml                   # Kustomization CR: ./apps · dependsOn: infrastructure
│
├── infrastructure/                     # Platform components — owned by the platform
│   ├── cert-manager/                   # HelmRelease, ClusterIssuer, wildcard Certificate, namespace
│   ├── gateway/                        # Gateway: traefik-gateway · *.cralyx.com
│   ├── kyverno/                        # HelmRelease + namespace
│   ├── logging/                        # Loki HelmRelease + Alloy HelmRelease + namespace
│   ├── monitoring/                     # kube-prometheus-stack HelmRelease, Grafana TLS + routing
│   ├── traefik/                        # HelmChartConfig — enables Gateway API on k3s Traefik
│   └── kustomization.yaml
│
├── policies/
│   └── kyverno/
│       ├── disallow-latest.yaml        # Audit: no :latest image tags
│       ├── require-resources.yaml      # Audit: CPU + memory requests/limits required
│       └── kustomization.yaml
│
├── secrets/
│   ├── cloudflare/
│   │   └── api-token.yaml              # SOPS-encrypted Cloudflare API token
│   └── kustomization.yaml
│
└── .sops.yaml                          # Scope: secrets/**/*.yaml · fields: data + stringData only
```


---

## GitOps Workflow

### Reconciliation Dependency Chain

The reconciliation order is enforced through `dependsOn` declarations in the cluster-layer Kustomization files. Each layer sets `wait: true`, meaning Flux health-checks all resources before proceeding to dependents.

```mermaid
graph LR
    S["<b>secrets</b><br/>SOPS decrypt<br/>cloudflare-api-token"]
    I["<b>infrastructure</b><br/>cert-manager · Traefik<br/>Kyverno · Prometheus<br/>Loki · Alloy"]
    A["<b>apps</b><br/>vaultwarden<br/>linkding"]
    P["<b>policies</b><br/>disallow-latest<br/>require-resources"]

    S --> I
    I --> A
    I --> P
```

`wait: true` eliminates an entire class of race conditions. Without it: `ClusterIssuer` applied before cert-manager CRDs exist; `HTTPRoute` referencing a Gateway that hasn't started. Both have caused real incidents on this platform.

### Full Reconciliation Sequence

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'actorBkg': '#f0f9ff',
    'actorBorder': '#0284c7',
    'actorTextColor': '#0f172a',
    'actorLineColor': '#0284c7',
    'signalColor': '#0369a1',
    'signalTextColor': '#0f172a',
    'sequenceNumberColor': '#ffffff',
    'sequenceNumberBkg': '#0ea5e9'
  }
}}%%
sequenceDiagram
    autonumber
    
    %% External Participants
    participant Dev as Engineer
    participant Git as Git (main)
    
    %% Cluster Boundary Grouping
    box rgb(224, 242, 254) "Kubernetes Cluster Boundary"
        participant Src as Source Controller
        participant Kust as Kustomize Controller
        participant SOPS as SOPS / Age
        participant Kyverno as Kyverno Webhook
        participant K8s as Kubernetes API
    end

    %% Flow Execution
    Dev->>Git: git push
    Src->>Git: Poll every 10 minutes
    Src->>Src: Detect revision change, fetch artifact
    Src->>Kust: Artifact ready
    Kust->>SOPS: Decrypt secrets/ manifests
    SOPS-->>Kust: Plaintext Secrets
    Kust->>Kyverno: Submit resources for admission
    Kyverno-->>Kust: Admit (or record Audit violation)
    Kust->>K8s: Apply resources
    K8s-->>Dev: Reconciliation complete (~10m)
```

`prune: true` is set on every Kustomization. Resources removed from Git are automatically deleted from the cluster on the next cycle, no manual cleanup, no resource accumulation over time.

---
## Image Automation

The image automation pipeline creates a fully closed update loop. New container versions are discovered, policy-evaluated, committed to Git, and deployed without any human action.

### The Setter Annotation

The `ImageUpdateAutomation` controller uses the `Setters` strategy, scanning the repository for structured marker comments and rewriting the image tag in-place:

```yaml
# apps/vaultwarden/deployment.yaml
image: vaultwarden/server:1.36.0 # {"$imagepolicy": "flux-system:vaultwarden"}

# apps/linkding/deployment.yaml
image: sissbruecker/linkding:1.45.0 # {"$imagepolicy": "flux-system:linkding"}
```

### Policy Scope

| App | Image | Policy range | Auto-deploys | Requires manual review |
|---|---|---|---|---|
| Vaultwarden | `vaultwarden/server` | `1.x` | Any `1.y.z` release | `2.0.0` and above |
| Linkding | `sissbruecker/linkding` | `1.x` | Any `1.y.z` release | `2.0.0` and above |

The `1.x` upper bound is the deliberate control point. Major version bumps may carry breaking schema changes or data migration requirements. They require a human to review the changelog, update the policy range, and explicitly accept the upgrade.

###### Relationship to Kyverno's `disallow-latest`

Image automation and the Kyverno `disallow-latest` policy are mutually reinforcing. Automation ensures all production images carry pinned semver tags, exactly what the policy requires. Kyverno acts as an independent, synchronous check: if a `:latest` tag were ever committed manually, it generates an audit violation in `PolicyReport` before it could be acted on.

---
## Application Delivery

Applications are defined as plain Kubernetes manifests, no Helm chart wrapping. This keeps the resource model transparent and avoids rendering indirection for workloads with minimal configuration surface.

### Vaultwarden

```yaml
image: vaultwarden/server:1.36.0  # {"$imagepolicy": "flux-system:vaultwarden"}
resources:
  requests: { cpu: 50m,  memory: 64Mi  }
  limits:   { cpu: 250m, memory: 256Mi }
env:
  - name: SIGNUPS_ALLOWED
    value: "false"                      # closed instance; registration disabled
  - name: DOMAIN
    value: "https://vault.cralyx.com"
```

Data persists to a PVC mounted at `/data`. k3s's `local-path` StorageClass provisions it as a host-path volume.

### Linkding

```yaml
image: sissbruecker/linkding:1.45.0  # {"$imagepolicy": "flux-system:linkding"}
resources:
  requests: { cpu: 50m,  memory: 128Mi }
  limits:   { cpu: 500m, memory: 512Mi }
```

Data persists at `/etc/linkding/data`. The Service maps port `80` to the container's port `9090`.

### What Both Workloads Have in Common

Both satisfy every active Kyverno policy: pinned image tags and explicit CPU/memory `requests` and `limits`. This is not incidental, it reflects deliberate alignment between policy definitions and workload configuration. Neither application defines an `Ingress` resource; routing is handled at the Gateway API layer via `HTTPRoute` resources, keeping routing concerns out of application manifests.

---

## Networking Architecture

### Traffic Flow

```mermaid
graph TB
    %% --- Style Definitions ---
    classDef client fill:#f8f9fa,stroke:#adb5bd,stroke-width:2px,color:#212529
    classDef cloudflare fill:#f6821f,stroke:#d35400,stroke-width:2px,color:#ffffff
    classDef gateway fill:#24a1de,stroke:#1a7cae,stroke-width:2px,color:#ffffff
    classDef cert fill:#2ecc71,stroke:#27ae60,stroke-width:2px,color:#ffffff
    classDef route fill:#9b59b6,stroke:#8e44ad,stroke-width:2px,color:#ffffff
    classDef service fill:#f1c40f,stroke:#d68910,stroke-width:2px,color:#212529

    %% --- Nodes ---
    Browser["Client Browser"]:::client
    CF["Cloudflare\nDNS Proxy — *.cralyx.com → cluster IP"]:::cloudflare
    GW["Traefik Gateway — traefik-gateway\nnamespace: cert-manager\nport: 8443 · TLS Terminate\nhostname: *.cralyx.com"]:::gateway
    CERT["Secret: wildcard-cralyx-com-tls\nLet's Encrypt *.cralyx.com"]:::cert
    VW_ROUTE["HTTPRoute: vault.cralyx.com"]:::route
    LD_ROUTE["HTTPRoute: linkding.cralyx.com"]:::route
    VW_SVC["Service: vaultwarden :80"]:::service
    LD_SVC["Service: linkding :80"]:::service

    %% --- Connections ---
    Browser -->|"HTTPS :443"| CF
    CF -->|"proxy → :8443"| GW
    GW -.->|"TLS cert"| CERT
    GW --> VW_ROUTE --> VW_SVC
    GW --> LD_ROUTE --> LD_SVC
```

### Gateway API — Infrastructure vs Application Ownership

The move from `Ingress` to Gateway API is an ownership boundary, not just an API upgrade. Platform engineers own the `Gateway`  what hostnames are exposed, on what ports, with what TLS configuration. Application owners own the `HTTPRoute`  how traffic to their hostname reaches their Service. Neither can override the other. The API enforces the boundary.

`allowedRoutes.namespaces.from: All` is correct for a single-operator platform where every namespace is trusted. A multi-team environment would use `from: Selector` with namespace label constraints.

**A non-obvious constraint:** k3s's Traefik maps named entrypoints (`web`, `websecure`) to internal ports `8000` and `8443`  not the external service ports `80` and `443`. Gateway API listeners must use the **internal entrypoint port**. Using `port: 443` in the listener spec causes listener validation to fail because Traefik's Gateway controller matches listeners by entrypoint port, not service port. This is not prominently documented and was identified through Traefik controller log inspection. The correct configuration is `port: 8443` in the Gateway listener with `hostPort: 443` in the `HelmChartConfig` the external port and the listener port are intentionally different.

`hostPort` binds Traefik directly to the node's network interface, bypassing the need for a `LoadBalancer` Service or MetalLB.

---

## Observability Stack

```mermaid
graph TD
    PODS[Application Pods<br/>Vaultwarden · Linkding] -->|stdout / stderr| ALLOY[Alloy DaemonSet<br/>log collection]
    ALLOY -->|LogQL push| LOKI[Loki<br/>log storage]
    LOKI -->|LogQL query| GRAF[Grafana]

    PODS -->|/metrics endpoint| PROM[Prometheus<br/>metrics scrape]
    K8S[Kubernetes Components<br/>kube-state-metrics · node-exporter] --> PROM
    PROM -->|PromQL query| GRAF

    GRAF -->|HTTPRoute| GW[Traefik Gateway]
    GW --> BROWSER[Operator Browser]

    style PODS fill:#2c3e50,color:#fff
    style ALLOY fill:#e65100,color:#fff
    style LOKI fill:#1565c0,color:#fff
    style PROM fill:#e65100,color:#fff
    style GRAF fill:#e65100,color:#fff
```

**Prometheus** (via `kube-prometheus-stack`) scrapes metrics from all platform components Flux controllers, Traefik, Kyverno, node-level metrics via `node-exporter` using `PodMonitor` and `ServiceMonitor` resources for service discovery.

**Grafana Alloy** runs as a `DaemonSet`, tailing container logs from every pod and enriching streams with Kubernetes metadata (`namespace`, `pod`, `container`) before forwarding to Loki. These labels become the primary dimensions for log queries, which maps naturally to how Kubernetes workloads are reasoned about during incidents.

**Loki** stores logs in a label-indexed format. It does not perform full-text indexing, the query model is optimised for the label-scoped filtering that Kubernetes log analysis naturally uses.

**Grafana** is the unified observability frontend with Prometheus and Loki configured as data sources. The logging and monitoring stacks are kept in separate namespaces (`logging` and `monitoring`) to allow independent lifecycle management, upgrading Loki does not require touching the Prometheus stack.

---

## Security Architecture

```mermaid
graph TD
    DEV[Developer] -->|git commit with SOPS-encrypted Secret| GH[GitHub<br/>encrypted ciphertext at rest]
    GH -->|Source Controller fetches| SC[GitRepository Artifact<br/>still encrypted]
    SC -->|Kustomize Controller reads| FLUX[Flux Decryption<br/>age private key from sops-age Secret]
    FLUX -->|plaintext manifest| API[Kubernetes API Server]
    API --> SECRET[Kubernetes Secret<br/>etcd at rest]
    SECRET -->|mounted or env| CERT[cert-manager<br/>Cloudflare DNS01]
    SECRET -->|mounted or env| APP[Application Workloads]

    style GH fill:#24292e,color:#fff
    style FLUX fill:#6a1b9a,color:#fff
    style SECRET fill:#4a235a,color:#fff
```


The `.sops.yaml` scopes encryption precisely:

```yaml
creation_rules:
  - path_regex: secrets/.*\.yaml           # only files under secrets/
    encrypted_regex: '^(data|stringData)$'  # only Secret value fields, not metadata
    age: age1n7qx554qx2h7tlascwe4nlsp0lh04pr5dlzwrx23vhzp7jada4tqny07rc
```

Only `data` and `stringData` fields are encrypted. The rest of each Secret manifest `name`, `namespace`, `labels` remains readable in plaintext, keeping the repository auditable without requiring decryption. The Age private key is stored as a Kubernetes Secret in `flux-system` during bootstrap and is the only credential that cannot be sourced from this repository.

SOPS decryption is explicitly scoped to the `secrets` Kustomization only:

```yaml
# clusters/production/secrets.yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

No other Kustomization has `decryption` configured. Decryption is an explicit, auditable capability and not a cluster-wide default.

### Admission Control: Kyverno

**`disallow-latest-tag`** rejects any Pod where a container uses the `:latest` image tag. Mutable tags make the running image unauditable, you cannot determine what code is running from the manifest, and you cannot roll back to a known state. All platform workloads satisfy this policy because image automation enforces pinned semver tags as process.

**`require-resource-requests-limits`** rejects any Pod where a container omits CPU or memory `requests` and `limits`. Without limits, a misbehaving container can exhaust node resources and starve every other workload. On a single-node cluster this means full platform outage.

Both policies use `validationFailureAction: Audit` with `background: true`. Audit mode records violations in `PolicyReport` resources without blocking admission, the correct posture while infrastructure Helm charts are being audited for compliance. `background: true` re-evaluates existing resources continuously.

```bash
# Inspect current policy compliance across all namespaces
kubectl get policyreport -A
kubectl describe clusterpolicyreport
```

### TLS and Application Security

All public endpoints are HTTPS-only. No HTTP application endpoint is exposed to the public internet. The wildcard certificate is issued and renewed by cert-manager automatically. Vaultwarden is configured with `SIGNUPS_ALLOWED=false` the instance accepts no new registrations and serves a known, fixed set of users.

---
## Engineering Decisions and Tradeoffs

### k3s Built-in Traefik via `HelmChartConfig`

**Decision**: Configure k3s's bundled Traefik via `HelmChartConfig` rather than disabling it and deploying a standalone Flux `HelmRelease`.

**Rationale**: k3s runs its own internal Helm controller managing the Traefik release. A Flux `HelmRelease` targeting the same release name causes ownership conflicts, both controllers attempt to reconcile the same objects. Working with k3s's grain via `HelmChartConfig` avoids the conflict and reduces the total number of controllers in the cluster.

**Tradeoff**: Traefik upgrades are tied to k3s releases rather than independently managed. `failurePolicy: reinstall` is a blunt recovery mechanism, a misconfigured value triggers a full reinstall, not a rollback. A dedicated HelmRelease with `upgrade.remediation.strategy: rollback` would handle failure more gracefully at the cost of the ownership conflict described above.

### Wildcard Certificate vs Per-Application Certificates

**Decision**: A single `*.cralyx.com` wildcard certificate rather than per-app `Certificate` resources.

**Rationale**: Adding a new subdomain requires only an `HTTPRoute`. No cert-manager interaction, no ACME rate limit exposure, one secret to manage and renew. During rapid iteration, per-app certs introduce friction and rate limit risk without meaningful benefit.

**Tradeoff**: Certificate compromise affects all subdomains simultaneously. The wildcard secret's placement in `cert-manager` constrains Gateway placement, the Gateway must be co-located with its TLS secret or use a cross-namespace `ReferenceGrant`. Per-app certs would provide blast-radius isolation at the cost of this operational simplicity.

### SOPS/Age vs External Secrets Operator

**Decision**: SOPS-encrypted secrets committed to Git rather than pulling from an external vault.

**Rationale**: No runtime dependency on an external service. The cluster is fully self-contained reconstructable offline from this repository and the Age key alone. Secret history is auditable in Git.

**Tradeoff**: Rotating the Age key requires re-encrypting every file matching the `creation_rules` path regex, a bulk operation rather than a single key rotation event. No per-access audit log at the secret read level. External Secrets Operator with HashiCorp Vault would provide both at the cost of an additional operational dependency.

### Image Automation Committing Directly to `main`

**Decision**: `ImageUpdateAutomation` pushes updated image tags directly to `main` without a pull request gate.

**Rationale**: For patch and minor releases within `1.x`, automated updates are safe by policy definition. Requiring a human-approved PR for every image update creates toil that defeats the automation's purpose.

**Tradeoff**: No review gate between a published Docker Hub tag and a running container. A compromised upstream image publishing a `1.x`-compliant tag deploys within ~1 hour of publication. Mitigated by the `1.x` upper bound and the established security track records of both projects. Cosign image verification would close this gap with a cryptographic guarantee.

### Kyverno Policies in `Audit` Mode

**Decision**: Both `ClusterPolicy` resources use `validationFailureAction: Audit` rather than `Enforce`.

**Rationale**: Platform components deployed by Helm charts may not satisfy every policy by default. `Enforce` mode during bootstrap could block infrastructure from coming up. `Audit` provides visibility without instability during the build phase.

**Tradeoff**: Non-compliant pods from Helm charts can run without being blocked. The intended path is to audit violations, adjust chart values or add exclusions, then promote each policy to `Enforce` incrementally.

---
## Known Gaps and Honest Assessment

| Gap                                        | Impact                                                        | Planned Resolution                                                                |
| ------------------------------------------ | ------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Kyverno policies in `Audit`, not `Enforce` | Non-compliant Helm chart pods run unblocked                   | Audit chart defaults; promote policies to `Enforce` incrementally                 |
| `allowedRoutes.namespaces.from: All`       | Any namespace can attach HTTPRoutes to the Gateway            | Acceptable now; switch to `Selector` if untrusted namespaces are added            |
| Image automation commits direct to `main`  | No review gate on automated tag updates                       | Acceptable for `1.x` range; Cosign verification would add supply chain confidence |
| Single-node cluster                        | Node failure = full platform outage                           | Acceptable for homelab; multi-node k3s requires etcd HA                           |
| No image signature verification            | Semver-compliant but compromised image would be auto-deployed | Add Cosign + Kyverno `verifyImages` rule                                          |
| No PVC backup                              | Vaultwarden and Linkding data has no offsite copy             | Add Velero with an S3-compatible backend                                          |








