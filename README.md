# FalconAKSAutomationWithHelm

# CrowdStrike Falcon Deployment Guide for Kubernetes

This guide provides instructions for deploying CrowdStrike Falcon components in a Kubernetes environment, specifically on Azure Kubernetes Service (AKS).

## Prerequisites

- CrowdStrike API credentials (Client ID and Secret)
- CrowdStrike Customer ID (CID)
- Azure subscription and AKS cluster
- `kubectl`, `helm`, `az` CLI, and `jq` installed
- Latest pull script 

FalconNodeSensor CR Configuration using CrowdStrike API Keys
Important

To start the FalconNodeSensor installation using CrowdStrike API Keys to allow the operator to determine your Falcon Customer ID (CID) as well as pull down the CrowdStrike Falcon Sensor container image, please create the following FalconNodeSensor resource to your cluster. You will need to provide CrowdStrike API Keys and CrowdStrike cloud region for the installation. It is recommended to establish new API credentials for the installation at https://falcon.crowdstrike.com/support/api-clients-and-keys, required permissions are:

Falcon Images Download: Read

Sensor Download: Read

## Environment Setup

Set the following environment variables with your specific values:

```bash
export FALCON_CLIENT_ID=<YOUR_CLIENT_ID>
export FALCON_CLIENT_SECRET=<YOUR_CLIENT_SECRET>
export FALCON_CID=<YOUR_CID>
export SENSOR_REGISTRY=registry.crowdstrike.com/falcon-sensor/release/falcon-sensor
export IARREGISTRY=registry.crowdstrike.com/falcon-imageanalyzer/release/falcon-imageanalyzer
export IARREGISTRY=registry.crowdstrike.com/falcon-imageanalyzer/us-1/release/falcon-imageanalyzer
export KACREGISTRY=registry.crowdstrike.com/falcon-kac/us-1/release/falcon-kac
export AZURESUBSCRIPTION=<YOUR_AZURE_SUBSCRIPTION_ID>
export CLUSTERRESOURCEGROUP=<YOUR_RESOURCE_GROUP>
export CLUSTERNAME=<YOUR_CLUSTER_NAME>
```

Replace the placeholders with your actual values:
- `<YOUR_CLIENT_ID>`: Your CrowdStrike API client ID
- `<YOUR_CLIENT_SECRET>`: Your CrowdStrike API client secret
- `<YOUR_CID>`: Your CrowdStrike customer ID
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
unset PULL_TOKEN
# Get sensor tags
export PULL_TOKEN=$(bash ./falcon-container-sensor-pull.sh -u $FALCON_CLIENT_ID -s $FALCON_CLIENT_SECRET -t falcon-kac --get-pull-token)
export SENSOR=$(bash ./falcon-container-sensor-pull.sh -u $FALCON_CLIENT_ID -s $FALCON_CLIENT_SECRET -t falcon-sensor --list-tags)
export LATEST_SENSOR_TAG=$(echo "$SENSOR" | jq -r '.tags | sort_by(split("-")[0] | split(".") | map(tonumber)) | last')

# Install Falcon Sensor
helm upgrade --install falcon-sensor crowdstrike/falcon-sensor \
  -n falcon-system --create-namespace \
  --set falcon.cid=$FALCON_CID \
  --set node.image.repository=$SENSOR_REGISTRY \
  --set node.image.tag=$LATEST_SENSOR_TAG \
  --set node.image.registryConfigJSON=$PULL_TOKEN

# Verify Sensor deployment
kubectl get pods -n falcon-system

```

## 2. Kubernetes Admission Controller (KAC) Deployment

Deploy the CrowdStrike Kubernetes Admission Controller:

```bash
export PULL_TOKEN=$(bash ./falcon-container-sensor-pull.sh -u $FALCON_CLIENT_ID -s $FALCON_CLIENT_SECRET -t falcon-kac --get-pull-token)
export KAC=$(bash ./falcon-container-sensor-pull.sh -u $FALCON_CLIENT_ID -s $FALCON_CLIENT_SECRET -t falcon-kac --list-tags)
export LATEST_KAC_TAG=$(echo "$KAC" | jq -r '.tags | sort_by(split("-")[0] | split(".") | map(tonumber)) | last')

# Add CrowdStrike Helm repo
helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
helm repo update

# Connect to AKS cluster
az account set --subscription $AZURESUBSCRIPTION
az aks get-credentials --resource-group $CLUSTERRESOURCEGROUP --name $CLUSTERNAME

# Install Falcon KAC
helm upgrade --install falcon-kac crowdstrike/falcon-kac \
  -n falcon-kac --create-namespace \
  --set falcon.cid=$FALCON_CID \
  --set image.repository=$KACREGISTRY \
  --set image.tag=$LATEST_KAC_TAG \
  --set image.registryConfigJSON=$PULL_TOKEN

# Verify KAC deployment
kubectl get pods -n falcon-kac
```

## 3. Image Analyzer (IAR) Deployment

Deploy the CrowdStrike Image Analyzer:

```bash
# Get the latest IAR tag and pull token
unset PULL_TOKEN
export IAR=$(bash ./falcon-container-sensor-pull.sh -u $FCS_CLIENT_ID -s $FALCON_CLIENT_SECRET --type falcon-imageanalyzer --list-tags)
export LATEST_IAR_TAG=$(echo "$IAR" | jq -r '.tags | .[-1]')
export PULL_TOKEN=$(bash ./falcon-container-sensor-pull.sh -u $FCS_CLIENT_ID -s $FALCON_CLIENT_SECRET --type falcon-imageanalyzer  --get-pull-token)

# Install Falcon Image Analyzer
helm upgrade --install falcon-image-analyzer crowdstrike/falcon-image-analyzer \
  -n falcon-image-analyzer --create-namespace \
  --set deployment.enabled=true \
  --set crowdstrikeConfig.cid=$FALCON_CID \
  --set crowdstrikeConfig.clientID=$FALCON_CLIENT_ID \
  --set crowdstrikeConfig.clientSecret=$FALCON_CLIENT_SECRET \
  --set image.repository=$IARREGISTRY \
  --set image.tag=$LATEST_IAR_TAG \
  --set image.registryConfigJSON=$PULL_TOKEN
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
