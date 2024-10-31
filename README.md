## Tutorial: Deploy GitHub Actions Runner Controller on AKS Using ACR

### Overview

This tutorial demonstrates how to:
1. Create and configure an Azure Container Registry (ACR).
2. Import the GitHub Actions Runner Controller Helm chart into ACR.
3. Attach the ACR to an AKS cluster and deploy the controller from ACR.

### Prerequisites
- **Azure CLI**, **kubectl**, and **Helm** are installed on your machine or Azure Cloud Shell.
- An AKS cluster is set up, or create one as detailed below.
- A resource group for the ACR (separate from or the same as the AKS resource group).

### Step 1: Define Environment Variables

Define variables for your setup. Replace values as needed.

```bash
# Core Variables
$AKS_RESOURCE_GROUP="aksRG"                                              # AKS resource group name
$LOCATION="CentralUS"                                                    # Region
$AKS_CLUSTER_NAME="aksCluster"                                           # AKS cluster name
$ACR_NAME="hcats0503022020"                                              # ACR name (must be unique across Azure)
$ACR_RESOURCE_GROUP="aksRG"                                              # ACR resource group name
$GITHUB_CONFIG_URL="https://github.com/hariscats/actions-aks-demo"       # GitHub repo/org URL for the controller

# Namespace Variables
$NAMESPACE_ARC_CONTROLLER="arc-controller-ns"     # Namespace for ARC controller
$NAMESPACE_ARC_RUNNERS="arc-runners-ns"           # Namespace for ARC runners
$ARC_CONTROLLER_NAME="arc-controller"             # Helm release name for ARC controller
$ARC_RUNNER_SCALESET_NAME="arc-runner-set"        # Helm release name for runner scale set
$ARC_RUNNER_GITHUB_SECRET_NAME="github-secret"    # Name of the secret for GitHub credentials
```

Create resource group.

```bash
az group create --name $AKS_RESOURCE_GROUP --location $LOCATION
```

### Step 2: Create an Azure Container Registry (ACR)

Create an ACR in the specified resource group.

```bash
az acr create --resource-group $AKS_RESOURCE_GROUP --name $ACR_NAME --sku Standard
```

Create AKS cluster.

```bash
az aks create `
    --resource-group $AKS_RESOURCE_GROUP `
    --name $AKS_CLUSTER_NAME `
    --os-sku AzureLinux `
    --node-count 1 `
    --enable-cluster-autoscaler `
    --min-count 1 `
    --max-count 3 `
    --node-vm-size Standard_D4s_v5 `
    --max-pods 100 `
    --network-plugin azure `
    --generate-ssh-keys
```

### Step 3: Attach ACR to AKS Cluster

For AKS to pull images from ACR, grant it the required permissions.

1. **For AKS with Managed Identity**:

   ```bash
   az aks update -n $AKS_CLUSTER_NAME -g $AKS_RESOURCE_GROUP --attach-acr $ACR_NAME
   ```

2. **For AKS with Service Principal**:

   For older clusters that use a Service Principal (SP), get the SP client ID and assign the `AcrPull` role to it:

   ```bash
   # Get the AKS Service Principal ID
   AKS_SP_ID=$(az aks show --resource-group $AKS_RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query "servicePrincipalProfile.clientId" --output tsv)

   # Grant AKS Service Principal access to ACR
   ACR_ID=$(az acr show --name $ACR_NAME --resource-group $ACR_RESOURCE_GROUP --query "id" --output tsv)
   az role assignment create --assignee $AKS_SP_ID --role AcrPull --scope $ACR_ID
   ```

### Step 4: Import the GitHub Actions Runner Helm Chart to ACR

Import the GitHub Actions Runner Controller Helm chart from GitHub’s container registry to your ACR.

```bash
az acr import --name $ACR_NAME --source ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller:0.9.0 --image actions-runner-controller/gha-runner-scale-set-controller:0.9.0
```

To verify that the chart was imported successfully:

```bash
az acr repository show --name $ACR_NAME --repository actions-runner-controller/gha-runner-scale-set-controller
```

### Step 5: Configure `kubectl` to Access the cluster

Configure `kubectl` to use AKS credentials:

```bash
az aks get-credentials --resource-group $AKS_RESOURCE_GROUP --name $AKS_CLUSTER_NAME
```

Verify the connection:

```bash
kubectl get nodes
```

### Step 6: Deploy the ARC Runners Controller from ACR

1. **Create a Namespace for the Controller**:

   ```bash
   kubectl create namespace $NAMESPACE_ARC_CONTROLLER
   ```

2. **Deploy the ARC Controller from ACR using Helm**:

   Update the Helm command to point to the image repository in ACR:

   ```bash
   helm install $ARC_CONTROLLER_NAME `
       --namespace $NAMESPACE_ARC_CONTROLLER `
       --version "0.9.3" `
       --set image.repository="$ACR_NAME.azurecr.io/actions-runner-controller/gha-runner-scale-set-controller" `
       oci://$ACR_NAME.azurecr.io/actions-runner-controller/gha-runner-scale-set-controller
   ```

### Step 7: Deploy the ARC Runner Scale Set from ACR on AKS

1. **Create a Namespace for the Runners**:

   ```bash
   kubectl create namespace $NAMESPACE_ARC_RUNNERS
   ```

2. **Create the GitHub App Secret**:

   Populate these values with your GitHub App credentials:

   ```bash
   GITHUB_APP_ID="your_github_app_id"
   GITHUB_APP_INSTALLATION_ID="your_github_app_installation_id"
   GITHUB_APP_PRIVATE_KEY="your_github_app_private_key"

   kubectl create secret generic $ARC_RUNNER_GITHUB_SECRET_NAME \
       --namespace=$NAMESPACE_ARC_RUNNERS \
       --from-literal=github_app_id=$GITHUB_APP_ID \
       --from-literal=github_app_installation_id=$GITHUB_APP_INSTALLATION_ID \
       --from-literal=github_app_private_key="$GITHUB_APP_PRIVATE_KEY"
   ```

3. **Deploy the Runner Scale Set using Helm**:

   Deploy the ARC runner scale set from ACR, setting values for GitHub and the ACR image repository:

   ```bash
   helm install $ARC_RUNNER_SCALESET_NAME \
       --namespace $NAMESPACE_ARC_RUNNERS \
       --create-namespace \
       --set githubConfigUrl=$GITHUB_CONFIG_URL \
       --set githubConfigSecret=$ARC_RUNNER_GITHUB_SECRET_NAME \
       --set minRunners=1 \
       --set maxRunners=3 \
       --set runnerGroup=default \
       --version "0.9.3" \
       --set image.repository="$ACR_NAME.azurecr.io/actions-runner-controller/gha-runner-scale-set" \
       oci://$ACR_NAME.azurecr.io/actions-runner-controller/gha-runner-scale-set
   ```

### Step 8: Verify the Deployment

Confirm the pods are running successfully:

```bash
kubectl get pods -n $NAMESPACE_ARC_CONTROLLER
kubectl get pods -n $NAMESPACE_ARC_RUNNERS
```

### Step 9: Upgrade the ARC Runner Scale Set (if needed)

To upgrade the ARC Runner Scale Set, use the `helm upgrade` command. The below example sets new parameters:

```bash
helm upgrade --install $ARC_RUNNER_SCALESET_NAME \
    --namespace $NAMESPACE_ARC_RUNNERS \
    --set githubConfigUrl=$GITHUB_CONFIG_URL \
    --set githubConfigSecret=$ARC_RUNNER_GITHUB_SECRET_NAME \
    --set minRunners=1 \
    --set maxRunners=3 \
    --set runnerGroup=default \
    --version "0.9.3" \
    --set image.repository="$ACR_NAME.azurecr.io/actions-runner-controller/gha-runner-scale-set" \
    oci://$ACR_NAME.azurecr.io/actions-runner-controller/gha-runner-scale-set
```

### Step 10: Run GitHub Actions Workflows

With the GitHub Actions Runner configured, workflows targeting the runner in the AKS cluster can be executed directly from GitHub. Once a workflow is triggered, a pod in AKS will be dynamically created, execute the job, and then terminate when complete.
