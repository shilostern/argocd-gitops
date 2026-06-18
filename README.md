# GitOps Project: Declarative Infrastructure with ArgoCD, Kind, and Podman

This project demonstrates the implementation of GitOps methodology in a local environment. The solution utilizes ArgoCD to manage Kubernetes configurations, with all system resources managed as code (IaC) within this repository.

---

## 1. Project Overview
The solution is based on the declarative management of the `whoami` application. The workflow included setting up a local Kubernetes cluster using Kind (on the Podman engine) and configuring ArgoCD as a synchronization controller. I chose to implement the **Kustomize** deployment method to challenge myself and master new, advanced configuration management tools, as it was a methodology I hadn't used before. Every change committed to this repository is automatically reconciled to the desired state in the cluster by ArgoCD.

## 2. Repository Structure
The repository is designed using Kustomize principles, allowing for a clear separation between the base code and environment-specific adjustments (Overlays):

* **apps/whoami/base:** Contains the basic manifests common to all environments.
* **apps/whoami/environments:** Contains definitions specific to each environment (dev/prod).
* **argocd:** Contains the Application manifests defining what ArgoCD should synchronize.

**Why this structure?** To ensure flexibility (DRY - Don't Repeat Yourself). Changes in the base affect all environments, while Overlays allow for unique configurations without code duplication.

## 3. Deployment Workflow
GitOps was chosen for its total transparency (what is in Git is what runs in the cluster) and its ability to enable rapid recovery (Rollback) by reverting to a previous commit. The deployment process is based on pushing changes to the repo, automatic detection by ArgoCD, and applying them to the cluster without manual `kubectl` intervention.

## 4. Key Design Decisions
* **Podman over Docker:** Chosen for environment compatibility, which required specific adjustments to the container runtime.
* **Kustomize Deployment:** I consciously chose to use Kustomize as the deployment method specifically to gain experience and learn new methodologies for managing cluster resources.

## 5. The Learning Process
The learning path was gradual and hands-on, based on reading official ArgoCD and Kind documentation, troubleshooting network and runtime errors, and using AI as a partner to understand complex DevOps concepts in real-time.

## 6. Learning Approach
I approached this as a "Problem-Solving Journey." Instead of reading all documentation from scratch, I focused on resolving specific errors by breaking down complex commands (like `kubectl port-forward`) with the help of AI.

## 7. Technical Challenges & Solutions
During the setup of the GitOps environment and the deployment of the `whoami` app, I encountered several challenges:

* **Network Issues:** Pods entered an `ImagePullBackOff` state because the cluster was disconnected from the internet. The solution was to switch to an offline workflow using manual image loading.
* **Kind Load Limitations:** I encountered compatibility issues between Kind and Podman. The solution was to work directly with the Kubernetes container runtime.
* **Podman Machine Freeze:** Control Plane crashes required restarting the container and reconnecting the API server.
* **Image Naming Mismatch:** Manual imports created names like `localhost/...`, requiring manual updates in the Deployment files.

## 8. AI-Assisted Debugging (Prompts)
The questions that helped me solve the challenges included:
* "ArgoCD shows 'OutOfSync', how do I debug the generated manifest?"
* "I am getting ImagePullBackOff, how do I check if my container runtime in Kind can see my local images?"
* "How can I import a .tar file directly into Kind's containerd runtime?"

## 9. Future Improvements
* **Auto-Scaling:** Adding HPA (Horizontal Pod Autoscaler) to adjust pod count based on real-time load.
* **CI/CD Pipeline:** Integrating an automated process to build images (GitHub Actions) and update the repository automatically.

## 10. Production Environment Considerations
In a production scenario, I would implement:
* A secure private Container Registry.
* Monitoring with Prometheus and Grafana to track pod health.

## 11. Current Risks
* **Single Point of Failure:** Dependency on the local Podman Machine; its failure shuts down the cluster.
* **Security Gaps:** Manual image management lacks automated vulnerability scanning.

## 12. Scaling to 20 Clusters
Scaling from 2 to 20 clusters requires transitioning to a fleet-management approach:

ArgoCD ApplicationSets: I would implement ApplicationSets to automate application deployment across multiple clusters. This replaces manual application definitions with dynamic templates that automatically discover and provision resources on newly added clusters.

Hierarchical Environment Management: Utilizing a structured Kustomize hierarchy, I would manage configurations across diverse regions and environments (e.g., dev, staging, prod). This allows for a common base configuration while using overlays to specify environment-specific parameters, such as unique resource limits for clusters in different geographic regions.

Centralized RBAC: To maintain security across 20 clusters, I would implement centralized Role-Based Access Control (RBAC). Access would be managed via group-based policies—such as granting Developers read-only access, Integrators sync permissions in staging environments, and Cluster-Admins full administrative control—ensuring consistent permission enforcement across the entire cluster fleet.

---
