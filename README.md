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

$env:KIND_EXPERIMENTAL_PROVIDER="podman"
kind create cluster --name gitops-cluster
