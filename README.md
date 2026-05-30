# flux-gitops

A lightweight k3s cluster running on a Debian 13 OCI ARM VM, fully automated and managed through FluxCD for GitOps-based deployments. The platform uses Cilium (eBPF) for networking, Traefik for ingress routing, cert-manager with Cloudflare for automated TLS, and SOPS with Agefor encrypted secrets management.

