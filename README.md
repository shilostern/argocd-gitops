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
