# DevOps Infrastructure Setup Guide

This document walks through each step to provision the Azure resources and push your container images for the FastAPI backend and React frontend. All commands are shown explicitly (no environment variables) and include the exact secret values used.

---

## 1. Create Resource Group

```bash
az group create \
  --name rg-plantandgo \
  --location northeurope
```

---

## 2. Create Azure Container Registry (ACR)

```bash
az acr create \
  --resource-group rg-plantandgo \
  --name plantandgoacreu \
  --sku Basic \
  --admin-enabled true
```

---

## 3. Retrieve ACR Credentials

```bash
az acr credential show \
  --name plantandgoacreu \
  --resource-group rg-plantandgo \
  --query "{username:username, password:passwords[0].value}" \
  --output json
```

> **Output:**
>
> ```json
> {
>   "username": "plantandgoacreu",
>   "password": "ab0vDbaNYBMVuYuHrGVk+GVDORPRPMnrIrWQ5bqlUw+ACRCIVNLz"
> }
> ```

---

## 4. Register Resource Providers

### Cosmos DB Resource Provider

```bash
az provider register --namespace Microsoft.DocumentDB
```

_Wait \~1–2 minutes, or check:_

```bash
az provider show --namespace Microsoft.DocumentDB --query "registrationState"
```

### Container Instance Resource Provider

```bash
az provider register --namespace Microsoft.ContainerInstance
```

_Wait \~1–2 minutes, or check:_

```bash
az provider show --namespace Microsoft.ContainerInstance --query "registrationState"

```

---

## 5. Create Azure Cosmos DB (MongoDB API v4.0)

```bash
az cosmosdb create \
  --resource-group rg-plantandgo \
  --name plantandgomongodb \
  --kind MongoDB \
  --server-version 4.0 \
  --locations regionName=northeurope failoverPriority=0 isZoneRedundant=False \
  --default-consistency-level Eventual \
  --enable-automatic-failover false
```

---

## Log in to your ACR

```bash
docker login plantandgoacreu.azurecr.io \
  --username plantandgoacreu \
  --password ab0vDbaNYBMVuYuHrGVk+GVDORPRPMnrIrWQ5bqlUw+ACRCIVNLz
```

---

## Custom Mosquitto image

### Build and push

```bash
docker buildx build \
  --platform linux/amd64 \
  --tag plantandgoacreu.azurecr.io/mosquitto:latest \
  --push \
  ./mosquitto
```

### Delete existing broker (if any)

```bash
az container delete \
  --resource-group rg-plantandgo \
  --name mqtt-broker \
  --yes
```

### Create new MQTT broker instance

```bash
az container create \
  --resource-group rg-plantandgo \
  --name mqtt-broker \
  --image plantandgoacreu.azurecr.io/mosquitto:latest \
  --registry-login-server plantandgoacreu.azurecr.io \
  --registry-username plantandgoacreu \
  --registry-password 'ab0vDbaNYBMVuYuHrGVk+GVDORPRPMnrIrWQ5bqlUw+ACRCIVNLz' \
  --ports 1883 9001 \
  --dns-name-label mqtt-plantandgo-eu \
  --os-type Linux \
  --cpu 1 --memory 1
```

---

## 6. Retrieve Cosmos DB Connection String

```bash
az cosmosdb keys list \
  --name plantandgomongodb \
  --resource-group rg-plantandgo \
  --type connection-strings \
  --query "connectionStrings[0].connectionString" \
  --output tsv
```

> **Output (MONGO_URL):**
>
> ```
> mongodb://plantandgomongodb:4oZqxASbkVSBAP91GOmnZoKj12XYSrHz35ZRcbDFsH4Vg8Y0ddaCF8naseKG9ws1FUKPDH6uwC0LACDb8KgPgA==@plantandgomongodb.mongo.cosmos.azure.com:10255/?ssl=true&replicaSet=globaldb&retrywrites=false&maxIdleTimeMS=120000&appName=@plantandgomongodb@
> ```

---

## 7. Build & Push Backend Image

> Make sure you’re in your `SEP4-backend/` directory when you run these.
> **Choose a tag** (e.g. `sprint1`, `sprint2`, …):

```bash
docker buildx create --name mybuilder --use || true
docker buildx build \
  --platform linux/amd64 \
  --tag plantandgoacreu.azurecr.io/backend:${TAG} \
  --push \
  .
```

---

## 8. Deploy Backend to ACI

```bash
az container delete \
  --resource-group rg-plantandgo \
  --name ${TAG}-backend \
  --yes

az container create \
  --resource-group rg-plantandgo \
  --name ${TAG}-backend \
  --image plantandgoacreu.azurecr.io/backend:${TAG} \
  --cpu 1 \
  --memory 1.5 \
  --registry-login-server plantandgoacreu.azurecr.io \
  --registry-username plantandgoacreu \
  --registry-password "ab0vDbaNYBMVuYuHrGVk+GVDORPRPMnrIrWQ5bqlUw+ACRCIVNLz" \
  --dns-name-label ${TAG}-backend \
  --ip-address public \
  --os-type Linux \
  --ports 80 \
  --environment-variables \
      MONGO_URL="mongodb://plantandgomongodb:4oZqxASbkVSBAP91GOmnZoKj12XYSrHz35ZRcbDFsH4Vg8Y0ddaCF8naseKG9ws1FUKPDH6uwC0LACDb8KgPgA==@plantandgomongodb.mongo.cosmos.azure.com:10255/?ssl=true&replicaSet=globaldb&retrywrites=false&maxIdleTimeMS=120000&appName=@plantandgomongodb@" \
      MQTT_BROKER_URL="mqtt://mqtt-plantandgo-eu.northeurope.azurecontainer.io:1883"
```

---

## 9. Build & Push Frontend Image

```bash
docker buildx build \
  --platform linux/amd64 \
  --tag plantandgoacreu.azurecr.io/frontend:${TAG} \
  --build-arg REACT_APP_API_URL="http://fastapi-prod-instance.northeurope.azurecontainer.io:8000" \
  --push \
  ../frontend
```

---

## 10. Deploy Frontend to ACI

```bash
az container delete \
  --resource-group rg-plantandgo \
  --name ${TAG}-frontend \
  --yes

az container create \
  --resource-group rg-plantandgo \
  --name ${TAG}-frontend \
  --image plantandgoacreu.azurecr.io/frontend:${TAG} \
  --cpu 1 \
  --memory 1 \
  --registry-login-server plantandgoacreu.azurecr.io \
  --registry-username plantandgoacreu \
  --registry-password "ab0vDbaNYBMVuYuHrGVk+GVDORPRPMnrIrWQ5bqlUw+ACRCIVNLz" \
  --dns-name-label ${TAG}-frontend \
  --ip-address public \
  --os-type Linux \
  --ports 80
```
