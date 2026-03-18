# DevOps Exam Implementation (Azure DevOps + Angular + Docker + Kubernetes)

This project is ready for a full CI/CD flow using Azure DevOps with the professor GitHub repository as source.

## What is already implemented in this repo

- CI/CD pipeline: `azure-pipelines.yml`
- Docker image definition: `Dockerfile`
- Docker optimization: `.dockerignore`
- SonarQube scanner config: `sonar-project.properties`
- Kubernetes manifests:
  - `k8s/namespace.yaml`
  - `k8s/deployment.yaml`
  - `k8s/service.yaml`

## Required Azure DevOps service connections

Create these service connections with exactly these names used in pipeline:

1. `sc-sonarqube` (SonarQube Server)
2. `sc-acr` (Azure Container Registry)
3. `sc-aks` (Kubernetes / AKS)

> If your teacher also asks Azure RM explicitly, create it as well (`sc-azure-subscription`) even though this pipeline does not require it at runtime.

## Required agent pool

Create a self-hosted agent pool named:

- `SelfHostedLinux`

Agent must have:

- Docker
- Node.js 16+
- Trivy
- kubectl

## Required pipeline variables to set/update

In `azure-pipelines.yml`, update:

- `acrLoginServer: youracrname.azurecr.io`

You can also move this to pipeline variables in Azure DevOps UI.

## GitHub source strategy (your case)

You are stakeholder and using professor GitHub repo directly.

1. Azure DevOps -> Pipelines -> New Pipeline
2. Select **GitHub** as source
3. Select professor repo
4. Choose **existing azure-pipelines.yml**

No Azure Repos import is required.

## Branch strategy

- `dev` for development
- PR from `dev` to `main`
- Pipeline triggers:
  - CI on `dev` and `main`
  - CD deploy stage only when branch is `main`

## Pipeline flow implemented

1. Checkout source (git clone via pipeline checkout)
2. SonarQube prepare/analyze/publish
3. `npm ci` + `npm run build -- --configuration production`
4. Docker build
5. Trivy image scan (fails on HIGH/CRITICAL)
6. Docker push to ACR
7. Deploy to AKS:
   - apply namespace
   - apply deployment
   - apply service
   - rollout status check

## Kubernetes notes

`k8s/deployment.yaml` uses `__IMAGE__` placeholder.

Pipeline replaces this placeholder with the pushed image tag automatically before apply.

## Optional: quick local validation

- `npm ci --legacy-peer-deps`
- `npm run build -- --configuration production`
- `docker build -t test-angular:local .`

## Final architecture

Developer -> GitHub Repo -> Azure DevOps Pipeline -> Build -> Docker Image -> ACR -> AKS -> Service (LoadBalancer)
