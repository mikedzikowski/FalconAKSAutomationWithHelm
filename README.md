# FalconAKSAutomationWithHelm

# CrowdStrike Falcon Deployment Guide for Kubernetes

This guide provides instructions for deploying CrowdStrike Falcon components in a Kubernetes environment, specifically on Azure Kubernetes Service (AKS).

## Prerequisites

- CrowdStrike API credentials (Client ID and Secret)
- CrowdStrike Customer ID (CID)
- Azure subscription and AKS cluster
- `kubectl`, `helm`, `az` CLI, and `jq` installed

FalconNodeSensor CR Configuration using CrowdStrike API Keys
Important

To start the FalconNodeSensor installation using CrowdStrike API Keys to allow the operator to determine your Falcon Customer ID (CID) as well as pull down the CrowdStrike Falcon Sensor container image, please create the following FalconNodeSensor resource to your cluster. You will need to provide CrowdStrike API Keys and CrowdStrike cloud region for the installation. It is recommended to establish new API credentials for the installation at https://falcon.crowdstrike.com/support/api-clients-and-keys, required permissions are:

Falcon Images Download: Read

Sensor Download: Read

## Environment Setup

Set the following environment variables with your specific values:

```bash
export FCSCLIENTID=<YOUR_CLIENT_ID>
export FSCSECRET=<YOUR_CLIENT_SECRET>
export FCSCID=<YOUR_CID>
export SENSORREGISTRY=registry.crowdstrike.com/falcon-sensor/<REGION>/release/falcon-sensor
export IARREGISTRY=registry.crowdstrike.com/falcon-imageanalyzer/<REGION>/release/falcon-imageanalyzer
export KACREGISTRY=registry.crowdstrike.com/falcon-kac/<REGION>/release/falcon-kac
export AZURESUBSCRIPTION=<YOUR_AZURE_SUBSCRIPTION_ID>
export CLUSTERRESOURCEGROUP=<YOUR_RESOURCE_GROUP>
export CLUSTERNAME=<YOUR_CLUSTER_NAME>
```

Replace the placeholders with your actual values:
- `<YOUR_CLIENT_ID>`: Your CrowdStrike API client ID
- `<YOUR_CLIENT_SECRET>`: Your CrowdStrike API client secret
- `<YOUR_CID>`: Your CrowdStrike customer ID
- `<REGION>`: Your CrowdStrike region (e.g., us-1, eu-1)
- `<YOUR_AZURE_SUBSCRIPTION_ID>`: Your Azure subscription ID
- `<YOUR_RESOURCE_GROUP>`: The resource group containing your AKS cluster
- `<YOUR_CLUSTER_NAME>`: The name of your AKS cluster

## 1. Falcon Sensor Deployment

Deploy the CrowdStrike Falcon Sensor to your Kubernetes cluster:

```bash
# Download the pull script
curl -sSL -o falcon-container-sensor-pull.sh https://github.com/CrowdStrike/falcon-scripts/releases/latest/download/falcon-container-sensor-pull.sh
chmod 777 ./falcon-container-sensor-pull.sh

# Get the latest sensor tag and pull token
export SENSOR=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-sensor --list-tags)
export LATEST_SENSOR_TAG=$(echo "$SENSOR" | jq -r '.tags | sort | last')
export PULLTOKEN=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-sensor --get-pull-token)

# Connect to your AKS cluster
az account set --subscription $AZURESUBSCRIPTION
az aks get-credentials --resource-group $CLUSTERRESOURCEGROUP --name $CLUSTERNAME --overwrite-existing

# Deploy using Helm
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update
helm upgrade --install falcon-helm crowdstrike/falcon-sensor \
  -n falcon-system --create-namespace \
  --set falcon.cid=$FCSCID \
  --set node.image.repository=$SENSORREGISTRY \
  --set node.image.tag=$LATEST_SENSOR_TAG \
  --set node.image.registryConfigJSON=$PULLTOKEN
```

## 2. Kubernetes Admission Controller (KAC) Deployment

Deploy the CrowdStrike Kubernetes Admission Controller:

```bash
# Get the latest KAC tag
export KAC=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-kac --list-tags)
export LATEST_KAC_TAG=$(echo "$KAC" | jq -r '.tags | sort | last')

# Deploy using Helm
helm install falcon-kac crowdstrike/falcon-kac \
  -n falcon-kac --create-namespace \
  --set falcon.cid=$FCSCID \
  --set image.repository=$KACREGISTRY \
  --set image.tag=$LATEST_KAC_TAG \
  --set image.registryConfigJSON=$PULLTOKEN
```

## 3. Image Analyzer (IAR) Deployment

Deploy the CrowdStrike Image Analyzer:

```bash
# Get the latest IAR tag and pull token
export IAR=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-imageanalyzer --list-tags)
export LATEST_IAR_TAG=$(echo "$IAR" | jq -r '.tags | sort | last')
export PULLTOKEN=$(bash ./falcon-container-sensor-pull.sh -u $FCSCLIENTID -s $FSCSECRET --type falcon-imageanalyzer --get-pull-token)

# Deploy using Helm
helm upgrade --install iar crowdstrike/falcon-image-analyzer -n falcon-image-analyzer --create-namespace \
  --set deployment.enabled=true \
  --set crowdstrikeConfig.clusterName=$CLUSTERNAME \
  --set crowdstrikeConfig.clientID=$FCSCLIENTID \
  --set crowdstrikeConfig.clientSecret=$FSCSECRET \
  --set crowdstrikeConfig.agentRegion=us-1 \
  --set image.registryConfigJSON=$PULLTOKEN \
  --set image.repository=$IARREGISTRY \
  --set image.tag=$LATEST_IAR_TAG \
  --set crowdstrikeConfig.cid=$FCSCID
```

## Verification

After deployment, verify that all components are running correctly:

```bash
# Check Falcon Sensor
kubectl get pods -n falcon-system

# Check KAC
kubectl get pods -n falcon-kac

# Check Image Analyzer
kubectl get pods -n falcon-image-analyzer
```

# Falcon AKS Automation with Helm

This repository contains automation for deploying CrowdStrike Falcon components to Azure Kubernetes Service (AKS) using Helm.

## Prerequisites

- Azure CLI installed
- kubectl installed
- Helm 3.x installed
- Access to Azure Container Registry (ACR)
- CrowdStrike Registry credentials
- Azure Subscription
- AKS Cluster

## Components

### Falcon Container Sensor
Deploys the CrowdStrike Falcon Container Sensor to protect your AKS workloads.

### Falcon Self-Hosted Registry Assessment (SHRA)
Deploys CrowdStrike's SHRA solution to scan container images in your private registries.

## Deployment Steps

### Set Environment Variables
```
export AZURESUBSCRIPTION="<subscription_id>"
export CLUSTERRESOURCEGROUP="<resource_group>"
export CLUSTERNAME="<cluster_name>"
export DOCKER_USER="<docker_username>"
export DOCKER_PASSWORD="<docker_password>"
export DOCKER_EMAIL="<docker_email>"
```

### Deploy Falcon Container Sensor

1. Set Azure context and get AKS credentials:
```
az account set --subscription $AZURESUBSCRIPTION
az aks get-credentials --resource-group $CLUSTERRESOURCEGROUP --name $CLUSTERNAME --overwrite-existing
```

2. Create namespace and secret:
```
kubectl create namespace falcon-system
kubectl create secret docker-registry falcon-registry \
  --docker-server=registry.crowdstrike.com \
  --docker-username=$DOCKER_USER \
  --docker-password=$DOCKER_PASSWORD \
  --docker-email=$DOCKER_EMAIL \
  --namespace=falcon-system
```

3. Deploy using Helm:
```
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update
helm upgrade --install falcon-helm crowdstrike/falcon-sensor \
  --create-namespace \
  --namespace falcon-system \
  -f values_override.yaml
```

### Deploy Falcon SHRA

1. Create SHRA namespace:
```
kubectl create namespace falcon-self-hosted-registry-assessment
```

2. Create Docker registry secret:
```
kubectl create secret docker-registry acr-secret \
  --docker-server=$DOCKER_USER \
  --docker-username=$DOCKER_USER \
  --docker-password=$DOCKER_PASSWORD \
  --docker-email=$DOCKER_EMAIL \
  --namespace=falcon-self-hosted-registry-assessment
```

3. Deploy SHRA using Helm:
```
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update
helm upgrade --install \
  -f values_override.yaml \
  --create-namespace \
  --namespace falcon-self-hosted-registry-assessment \
  --wait falcon-shra crowdstrike/falcon-self-hosted-registry-assessment
```

## Configuration Files

### values_override.yaml
Create this file to override default Helm chart values. Example structure:

```
falcon:
  cid: "<your_CID>"
  apd: "<your_APD>"
  # Add additional configuration as needed
```

## Notes
- Ensure all prerequisites are met before deployment
- Allow approximately 30 seconds after secret creation before deploying components
- Verify your CrowdStrike credentials are valid
- Monitor pod status after deployment for any issues

## Troubleshooting

1. Check pod status:
```
kubectl get pods -n falcon-system
kubectl get pods -n falcon-self-hosted-registry-assessment
```

2. View pod logs:
```
kubectl logs <pod_name> -n falcon-system
kubectl logs <pod_name> -n falcon-self-hosted-registry-assessment
```

## Additional Resources
- [CrowdStrike Falcon Helm Charts](https://github.com/CrowdStrike/falcon-helm)
- [Falcon Container Sensor Documentation](https://falcon.crowdstrike.com/documentation/146/falcon-container-sensor-for-linux)
- [SHRA Documentation](https://falcon.crowdstrike.com/documentation/338/falcon-self-hosted-registry-assessment)

## Support
For issues with this automation, please open a GitHub issue.
For CrowdStrike product support, please contact CrowdStrike Support.

## Troubleshooting

If you encounter issues:

1. Check pod logs:
   ```bash
   kubectl logs -n <namespace> <pod-name>
   ```

2. Verify environment variables are correctly set
3. Ensure API credentials have appropriate permissions
4. Check Helm release status:
   ```bash
   helm list -n <namespace>
   ```

## Security Note

Keep your CrowdStrike API credentials secure. Never commit them to version control or share them in public forums.

---

For more information, refer to the [CrowdStrike documentation](https://falcon.crowdstrike.com/documentation/page/overview).
