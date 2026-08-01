# Shared DevOps Library

Reusable Helm charts and GitHub Actions CI/CD workflows used across all services in the gym chain system.

**Repository:** `github.com/gym-chain/common-devops`

---

## Repository Structure

```
common-devops/
├── .github/
│   └── workflows/
│       ├── go-ci.yml                 # Reusable Go service build/test
│       ├── java-ci.yml               # Reusable Java/Gradle service build/test
│       ├── docker-build.yml          # Reusable Docker build & push to GHCR
│       └── helm-deploy.yml           # Reusable Helm deploy to K8s
│
└── helm/
    ├── gym-service/                  # Umbrella/Generic Chart for services
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    │       ├── deployment.yaml
    │       ├── service.yaml
    │       ├── hpa.yaml
    │       ├── ingress.yaml
    │       ├── configmap.yaml
    │       ├── secrets.yaml
    │       └── _helpers.tpl
    │
    └── gym-infra/                    # Chart for local infrastructure deps
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── postgres.yaml
            ├── kafka.yaml
            ├── redis.yaml
            ├── cassandra.yaml
            └── yugabyte.yaml
```

---

## 1. Reusable Helm Chart (`gym-service`)

A single, highly configurable Helm chart is used to deploy any of the 9 microservices. Each service overrides this chart using its own `values.yaml`.

### `Chart.yaml`
```yaml
apiVersion: v2
name: gym-service
description: Generic Helm chart for Gym Chain microservices
type: application
version: 1.0.0
appVersion: "1.0"
```

### `values.yaml` (Defaults)
```yaml
replicaCount: 2

image:
  repository: ghcr.io/gym-chain/service-name
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  grpcPort: 50051
  httpPort: 8080
  metricsPort: 9090

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

env: []
secrets: []
```

---

## 2. Reusable GitHub Actions Workflows

Workflows are defined in `common-devops` and referenced by individual service pipelines in the main monorepo.

### Reusable Docker Build & Push (`.github/workflows/docker-build.yml`)
Pushes built images to GitHub Container Registry (GHCR).

```yaml
name: Reusable Docker Build & Push

on:
  workflow_call:
    inputs:
      service_name:
        required: true
        type: string
      context:
        required: true
        type: string
    secrets:
      GH_PAT:
        required: true

jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GH_PAT }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: ${{ inputs.context }}
          file: ${{ inputs.context }}/Dockerfile
          push: true
          tags: |
            ghcr.io/gym-chain/${{ inputs.service_name }}:${{ github.sha }}
            ghcr.io/gym-chain/${{ inputs.service_name }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Reusable Helm Deploy (`.github/workflows/helm-deploy.yml`)
Deploys services to K8s using Helm.

```yaml
name: Reusable Helm Deploy

on:
  workflow_call:
    inputs:
      service_name:
        required: true
        type: string
      environment:
        required: true
        type: string
      image_tag:
        required: true
        type: string
    secrets:
      KUBECONFIG:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Common DevOps Repo
        uses: actions/checkout@v4
        with:
          repository: gym-chain/common-devops
          path: devops

      - name: Set up Kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG }}" > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Install Helm
        uses: azure/setup-helm@v4

      - name: Deploy Service via Helm
        run: |
          helm upgrade --install ${{ inputs.service_name }} ./devops/helm/gym-service \
            --namespace gym-${{ inputs.environment }} \
            --create-namespace \
            --set image.tag=${{ inputs.image_tag }} \
            --values ./services/${{ inputs.service_name }}/values-${{ inputs.environment }}.yaml
```

---

## How Monorepo References Shared DevOps

In the main `gym-chain` repository, service workflows reference these templates directly.

### Example: `.github/workflows/ms-gym-member.yml`
```yaml
name: Member Service Pipeline

on:
  push:
    paths:
      - 'services/ms-gym-member/**'
    branches: [main, develop]

jobs:
  test:
    uses: gym-chain/common-devops/.github/workflows/java-ci.yml@main
    with:
      service_path: 'services/ms-gym-member'

  build:
    needs: test
    uses: gym-chain/common-devops/.github/workflows/docker-build.yml@main
    with:
      service_name: 'ms-gym-member'
      context: 'services/ms-gym-member'
    secrets:
      GH_PAT: ${{ secrets.GHCR_TOKEN }}

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/develop'
    uses: gym-chain/common-devops/.github/workflows/helm-deploy.yml@main
    with:
      service_name: 'ms-gym-member'
      environment: 'staging'
      image_tag: ${{ github.sha }}
    secrets:
      KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}
```
