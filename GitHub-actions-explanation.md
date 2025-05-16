# GitHub Actions Pipeline Setup & Explanation

This guide shows how GitHub Actions workflows are configure on our SEP4-frontend and SEP4-backend:

- Runs **CD** on every pull request against `main`
- Allows **manual deployment** by dispatch with a tag (e.g. `sprint1`, `sprint2`)
- Authenticates to Azure via a Service Principal (`AZURE_CREDENTIALS`)
- Builds and pushes to your Azure Container Registry (ACR)
- Deploys to Azure Container Instances (ACI) with a stable DNS name

---

## 1. Prerequisites

Before you start, make sure you’ve added the following **Organization-level** **Variables** and **Secrets** in **GitHub → Settings → Secrets and variables → Actions**:

### 1.1 Variables (non-secret)

| Name             | Value                        | Used for                                             |
| ---------------- | ---------------------------- | ---------------------------------------------------- |
| `RESOURCE_GROUP` | `rg-plantandgo`              | `--resource-group` flag for all `az` commands        |
| `ACR_NAME`       | `plantandgoacreu`            | Login name for your ACR                              |
| `ACR_SERVER`     | `plantandgoacreu.azurecr.io` | Full registry endpoint                               |
| `IMAGE_NAME`     | `backend`                    | Image repository name in ACR                         |
| `DNS_LABEL`      | `plantandgo-backend`         | Stable DNS name label for ACI (`plantandgo-backend`) |

### 1.2 Secrets (encrypted)

| Name                | Value                                                       | Used for                                      |
| ------------------- | ----------------------------------------------------------- | --------------------------------------------- |
| `AZURE_CREDENTIALS` | JSON from `az ad sp create-for-rbac … --sdk-auth`           | `azure/login@v1`                              |
| `ACR_PASSWORD`      | ACR admin password from `az acr credential show …`          | `docker/login-action@v2`                      |
| `MONGO_URL`         | Your CosmosDB (MongoDB API v4) connection string            | `az container create --environment-variables` |
| `MQTT_BROKER_URL`   | `mqtt://mqtt-plantandgo.northeurope.azurecontainer.io:1883` | `az container create --environment-variables` |

---

## Frontend Workflow File

```yaml
name: Deploy Frontend to Azure ACI

on:
  pull_request:
    branches:
      - main
  workflow_dispatch:
    inputs:
      tag:
        description: "Docker image tag (e.g. sprint1, sprint2)"
        required: true
        default: "test"

env:
  RESOURCE_GROUP: ${{ vars.RESOURCE_GROUP }}
  ACR_NAME: ${{ vars.ACR_NAME }}
  ACR_SERVER: ${{ vars.ACR_SERVER }}
  IMAGE_NAME: frontend
  BACKEND_URL: ${{vars.BACKEND_URL}}
  TAG: ${{ github.event_name == 'push' && startsWith(github.ref, 'refs/tags/') && github.ref_name || 'latest' }}

jobs:
  build-and-deploy-frontend:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Docker Login to ACR
        uses: docker/login-action@v2
        with:
          registry: ${{ env.ACR_SERVER }}
          username: ${{ env.ACR_NAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Build & Push Frontend Image
        uses: docker/build-push-action@v4
        with:
          context: .
          platforms: linux/amd64
          push: true
          tags: |
            ${{ env.ACR_SERVER }}/${{ env.IMAGE_NAME }}:${{ github.event.inputs.tag }}
          build-args: |
            REACT_APP_API_URL=http://${{ env.BACKEND_URL }}

      - name: Delete existing Frontend ACI (if any)
        run: |
          az container delete \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --name plantandgo-frontend \
            --yes || echo "No existing frontend to delete"

      - name: Deploy Frontend to Azure Container Instances
        run: |
          az container create \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --name plantandgo-frontend \
            --image ${{ env.ACR_SERVER }}/${{ env.IMAGE_NAME }}:${{ github.event.inputs.tag }} \
            --cpu 1 \
            --memory 1 \
            --registry-login-server ${{ env.ACR_SERVER }} \
            --registry-username ${{ env.ACR_NAME }} \
            --registry-password "${{ secrets.ACR_PASSWORD }}" \
            --dns-name-label plantandgo-frontend \
            --ip-address public \
            --os-type Linux \
            --ports 80
```

## Backend Workflow File

```yaml
name: Deploy Backend to Azure ACI

on:
  pull_request:
    branches:
      - main
  workflow_dispatch:
    inputs:
      tag:
        description: "Docker image tag (e.g. sprint1, sprint2)"
        required: true
        default: "test"

env:
  RESOURCE_GROUP: ${{ vars.RESOURCE_GROUP }}
  ACR_NAME: ${{ vars.ACR_NAME }}
  ACR_SERVER: ${{ vars.ACR_SERVER }}
  IMAGE_NAME: backend

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Docker Login to ACR
        uses: docker/login-action@v2
        with:
          registry: ${{ env.ACR_SERVER }}
          username: ${{ env.ACR_NAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Build & Push Backend Image
        uses: docker/build-push-action@v4
        with:
          context: .
          platforms: linux/amd64
          push: true
          tags: |
            ${{ env.ACR_SERVER }}/${{ env.IMAGE_NAME }}:${{ github.event.inputs.tag }}

      - name: Delete existing ACI (if any)
        run: |
          az container delete \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --name plantandgo-backend \
            --yes || echo "No existing container to delete"

      - name: Deploy to Azure Container Instances
        run: |
          az container create \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --name plantandgo-backend \
            --image ${{ env.ACR_SERVER }}/${{ env.IMAGE_NAME }}:${{ github.event.inputs.tag }} \
            --cpu 1 \
            --memory 1.5 \
            --registry-login-server ${{ env.ACR_SERVER }} \
            --registry-username ${{ env.ACR_NAME }} \
            --registry-password "${{ secrets.ACR_PASSWORD }}" \
            --dns-name-label plantandgo-backend \
            --ip-address public \
            --os-type Linux \
            --ports 80 \
            --environment-variables \
                MONGO_URL="${{ secrets.MONGO_URL }}" \
                MQTT_BROKER_URL="${{ secrets.MQTT_BROKER_URL }}"
```
