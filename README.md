# GitOps Deployment with ArgoCD, Kind, and Podman

This repository contains a production-ready GitOps implementation for deploying a lightweight application (`whoami`) across multiple logical environments using **ArgoCD**, **Kustomize**, and **Kind** running on a Windows environment powered by **Podman**.

---

## 🏛️ Repository Structure

The project follows a modular layout designed for high extensibility and strict compliance with DRY (Don't Repeat Yourself) principles:

```text
.
├── apps/
│   └── whoami/
│       ├── base/                      # Immutable core manifests
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── kustomization.yaml
│       ├── components/                # Reusable configuration patches
│       │   └── environment-variables/
│       │       └── kustomization.yaml
│       └── environments/              # Overlay definitions (Target environments)
│           ├── dev/
│           │   └── kustomization.yaml
│           └── prod/
│               └── kustomization.yaml
├── argocd/                            # ArgoCD Application declarations
│   ├── dev-app.yaml
│   └── prod-app.yaml
└── README.md

🚀 Deployment & Synchronization Process
1. Local Infrastructure Setup
The infrastructure is instantiated on Windows utilizing Podman as the container engine backend for Kind:

PowerShell
$env:KIND_EXPERIMENTAL_PROVIDER="podman"
kind create cluster --name gitops-cluster

2. ArgoCD Installation
ArgoCD is deployed directly onto the local cluster within its dedicated namespace:

Bash
kubectl create namespace argocd
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
3. Bootstrap Applications
The system uses declarative ArgoCD Application manifests to establish tracking against this Git repository:

Bash
kubectl apply -f argocd/dev-app.yaml
kubectl apply -f argocd/prod-app.yaml
Once applied, ArgoCD automatically triggers synchronization, instantiating the whoami-dev (1 replica) and whoami-prod (3 replicas) namespaces and workloads.

💡 Key Architectural Decisions
Kustomize Components over Standard Overlays: Instead of duplicating environment variables across manifests, a decoupled components/ layer was introduced. This ensures that modifications to global configurations are managed centrally.

Namespace Isolation via Targets: dev and prod targets leverage Kustomize namespace targets to guarantee rigid boundaries within a single cluster topology.

Declarative App-Manager: Environment applications are defined as version-controlled code (argocd/*.yaml), making disaster recovery as fast as a single kubectl apply execution.

🧠 Learning Journey & Challenges
Approach & Resources
Self-Guided Scope: Approached the assignment by thoroughly mapping out the relationship between Git changes and Kubernetes state coordination.

References: Utilized the official ArgoCD documentation, Kustomize structural patterns, and the CNCF GitOps standards landscape.

Encountered Challenges
Podman on Windows Integration: Combining Kind with Rootless Podman on a Windows host presents inherent context-sharing complexities. Ensuring persistence of the $env:KIND_EXPERIMENTAL_PROVIDER="podman" variable across shell reboots was resolved by establishing strict pre-flight check processes.

ArgoCD Validation Error: Addressed strict schema validation issues concerning namespace generation by refactoring standard syncPolicy fields into compliant syncOptions arrays.

🔮 Scalability & Future Enhancements
Production Hardening (If going live)
Secrets Security: Introduce SOPS or HashiCorp Vault integration via ArgoCD Sealed Secrets or Secret Manager plugins to stop plain-text parameters from being pushed to Git.

Ingress Control: Deploy an enterprise Ingress controller (such as Traefik or NGINX Ingress) backed by cert-manager for structured external traffic routing and SSL termination.

Observability: Bind Prometheus and Grafana alerts to the ArgoCD synchronization state to catch deployment drift instantly.

📈 Scaling from 2 to 20 Clusters
To seamlessly manage the scale up to 20 clusters over the coming year, the architecture will pivot to the following advanced paradigms:

Hub-and-Spoke Topology: Centralize a single control-plane ArgoCD deployment inside a dedicated Management Cluster (Hub). This hub securely authenticates against the remaining 19 remote workload clusters (Spokes) via encoded API Secrets, avoiding distributed configuration management.

ArgoCD ApplicationSets: Replace standalone application files with an ApplicationSet controller leveraging the Git Generator or Cluster Generator. When a new directory or cluster spec is checked into Git, ArgoCD will auto-generate the matching multi-cluster workload instantly without manual operations.

Access Control & RBAC Boundaries: Integrate corporate Identity Providers (OIDC via Okta or Keycloak) to orchestrate fine-grained ArgoCD Projects. Developers will be granted restricted read-only permissions for dev spaces, while senior engineers and automation pipelines maintain secure operational rights over prod environments.
