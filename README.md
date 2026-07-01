# learnhub-middleware-charts GitOps Repository

This repository contains the Kubernetes configurations for the LearnHub middleware services, managed via a GitOps workflow using ArgoCD. It is structured to handle multiple environments, applications, and configurations from a single source of truth: this Git repository.

## Core Concepts

This repository uses a sophisticated **App of Apps** pattern with ArgoCD's `ApplicationSet` resource. This allows for a highly automated and scalable way to manage application deployments.

1.  **GitOps**: This repository is the single source of truth. All changes to the desired state of the Kubernetes cluster (deployments, services, etc.) are made by pushing commits to this repository. ArgoCD ensures the cluster state matches the state defined here.

2.  **Root ApplicationSet (`bootstrap/envs-appset.yaml`)**: This is the entry point for ArgoCD. It scans the `envs` directory to discover which environments exist (e.g., `dev`, `test`, `prod`). For each environment it finds, it generates a parent ArgoCD Application. This parent application is responsible for deploying the common charts for that environment, such as `apps`, `networking`, and `security`.

3.  **Nested ApplicationSet (`charts/apps/templates/apps-appset.yaml`)**: This is the "App of Apps" part. The `apps` chart, when deployed by the root `ApplicationSet`, doesn't deploy a microservice directly. Instead, it deploys *another* `ApplicationSet` specific to that environment (e.g., `apps-appset-dev`). This nested `ApplicationSet` reads a `versions.yaml` file for the environment to discover which actual microservices (like `learnhub-api`) should be deployed.

This two-level structure provides a powerful separation of concerns: the root `ApplicationSet` manages environments, and the nested `ApplicationSet` manages the applications within each environment.

## Directory Structure

```
.
├── bootstrap/
│   └── envs-appset.yaml      # The root ApplicationSet that discovers environments.
├── charts/
│   ├── apps/                 # Helm chart for deploying microservices (contains the nested AppSet).
│   │   └── templates/
│   │       └── apps-appset.yaml
│   ├── networking/           # Helm chart for Istio resources (Gateway, VirtualService).
│   └── security/             # Helm chart for security policies (e.g., NetworkPolicy).
├── envs/
│   └── minikube/             # A region or cluster group (e.g., aws-us-east-1).
│       ├── dev/              # The 'dev' environment.
│       │   ├── apps/
│       │   │   └── versions.yaml # Lists the microservices and their versions for 'dev'.
│       │   └── values.yaml     # Overrides Helm values for the 'dev' environment.
│       ├── prod/             # The 'prod' environment.
│       │   └── ...
│       └── test/             # The 'test' environment.
│           └── ...
└── README.md                 # This file.
```

## How It Works: A Step-by-Step Workflow

1.  An administrator installs the root `ApplicationSet` from `bootstrap/envs-appset.yaml` into the ArgoCD namespace on the cluster.
2.  The root `ApplicationSet`'s generator scans the `envs/` directory and finds `minikube/dev`, `minikube/test`, and `minikube/prod`.
3.  For each environment, it generates a parent ArgoCD Application (e.g., `dev`).
4.  The `dev` Application syncs the charts specified in its `sources`, including the `apps` chart.
5.  When the `apps` chart is synced, its template (`apps-appset.yaml`) is rendered, creating a *new* `ApplicationSet` in the cluster named `apps-appset-dev`.
6.  This `apps-appset-dev` `ApplicationSet` then uses its own generator to find and read the file `envs/minikube/dev/apps/versions.yaml`.
7.  It finds an entry for the `learnhub` application in `versions.yaml`.
8.  It generates the final ArgoCD Application, `learnhub-dev`, which points to the `learnhub` application's source code repository.
9.  The `learnhub-dev` Application syncs, pulling the Helm chart from the application repository and deploying it to the `dev` namespace in the cluster.

## How to Use This Repository

### Prerequisites

*   A Kubernetes cluster.
*   ArgoCD installed on the cluster.
*   `kubectl` configured to communicate with your cluster.

### Onboarding a New Microservice

To deploy a new microservice (e.g., `new-service`) to the `dev` environment:

1.  **Add the service to the version manifest**:
    Open `envs/minikube/dev/apps/versions.yaml` and add an entry for your new service. The `version` field here corresponds to the **image tag** of your application's container.

    ```yaml
    - name: learnhub
      version: 1.0.3
    - name: new-service
      version: 1.0.0
    ```

2.  **Commit and Push**: Commit the change to Git and push it to the `main` branch.

ArgoCD will automatically detect the change and start the deployment process for `new-service` in the `dev` environment.

### Modifying Environment Configuration

To change a configuration value for an environment, you typically edit the `values.yaml` file for that environment.

For example, to re-enable the Istio networking components for the `dev` environment:

1.  **Edit the values file**:
    Open `envs/minikube/dev/values.yaml` and change `networking.enabled` to `true`.

    ```yaml
    hosts:
      - api-dev.learnhub.local
      - api-dev.learnhub.com
    networking:
      enabled: true # Was false
    ```

2.  **Commit and Push**: Commit and push the change. ArgoCD will sync the `dev` application and apply the changes, creating the `Gateway` and `VirtualService` resources.

### Role of Key Files

*   `bootstrap/envs-appset.yaml`: The master file that bootstraps all environments. You rarely need to touch this unless you change your directory structure for environments.
*   `charts/apps/templates/apps-appset.yaml`: Defines how microservices are discovered and deployed. You might edit this to change repository URLs or add new Helm value files.
*   `charts/networking/templates/`: Contains the Istio templates. We added an `{{if .Values.networking.enabled}}` block here to make them conditional.
*   `envs/**/versions.yaml`: The manifest for which applications and container image versions are deployed in a given environment. This is one of the most frequently edited files.
*   `envs/**/values.yaml`: Provides environment-specific overrides for Helm charts. Used for changing hostnames, replica counts, or enabling/disabling features like networking.
