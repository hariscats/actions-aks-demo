## Tutorial: Deploy GitHub Actions Runner Controller on AKS Using ACR

### Overview

This tutorial demonstrates how to:
1. Create and configure an Azure Container Registry (ACR).
2. Import GitHub Actions Runner and Runner Controller container images into ACR.
3. Clone, package, and push the Helm chart to ACR.
4. Deploy the GitHub Actions Runner Controller and Scale Set to AKS.
5. Test the setup by running a sample GitHub Actions workflow.

### Prerequisites
- **Azure CLI**, **kubectl**, and **Helm** installed locally or in Azure Cloud Shell.
- An AKS cluster set up, or instructions to create one are provided below.
- A resource group for the ACR, either separate or the same as the AKS resource group.

### Step 1: Define Environment Variables

Define variables for your setup and replace placeholder values as needed.

```powershell
# Core Variables
$AKS_RESOURCE_GROUP="aksRG"                         # AKS resource group name
$LOCATION="CentralUS"                               # Azure region
$AKS_CLUSTER_NAME="aksCluster"                      # AKS cluster name
$ACR_NAME="hcats0503022020"                         # ACR name (must be unique across Azure)
$ACR_RESOURCE_GROUP="aksRG"                         # ACR resource group name
$GITHUB_CONFIG_URL="https://github.com/hariscats/actions-aks-demo"  # GitHub repository or organization URL for the controller

# Namespace and Release Names
$NAMESPACE_ARC_CONTROLLER="arc-controller-ns"       # Namespace for the controller
$NAMESPACE_ARC_RUNNERS="arc-runners"                # Namespace for the runners
$ARC_CONTROLLER_NAME="scale-set-controller"         # Helm release name for the controller
$ARC_RUNNER_SCALESET_NAME="arc-runner-set"          # Helm release name for the runner scale set
$GITHUB_PAT="<PAT>"                                 # GitHub Personal Access Token
```

Create the resource group if it doesn’t exist:

```powershell
az group create --name $AKS_RESOURCE_GROUP --location $LOCATION
```

### Step 2: Create an Azure Container Registry (ACR)

Create an ACR in the specified resource group:

```powershell
az acr create --resource-group $AKS_RESOURCE_GROUP --name $ACR_NAME --sku Standard
```

Create the AKS cluster:

```powershell
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

For AKS to pull images from ACR, grant it the required permissions:

```powershell
az aks update -n $AKS_CLUSTER_NAME -g $AKS_RESOURCE_GROUP --attach-acr $ACR_NAME
```

### Step 4: Pull and Push Required Container Images to ACR

1. **Pull GitHub Actions Runner and Controller Images**:

   Pull the container images directly from GitHub Packages:
   - **Runner Scale Set Controller**: [GitHub Actions Runner Controller (v0.9.3)](https://github.com/actions/actions-runner-controller/pkgs/container/gha-runner-scale-set-controller/234741940?tag=0.9.3)
   - **Actions Runner**: [GitHub Actions Runner](https://github.com/actions/runner/pkgs/container/actions-runner)

   ```powershell
   docker pull ghcr.io/actions/gha-runner-scale-set-controller:0.9.3
   docker pull ghcr.io/actions/actions-runner:2.320.0
   ```

2. **Tag the Images for ACR**:

   Replace `<ACR_LOGIN_SERVER>` with your actual ACR login server, typically in the format `<acr-name>.azurecr.io`.

   ```powershell
   $ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer --output tsv)
   
   docker tag ghcr.io/actions/gha-runner-scale-set-controller:0.9.3 $ACR_LOGIN_SERVER/gha-runner-scale-set-controller:0.9.3
   docker tag ghcr.io/actions/actions-runner:2.320.0 $ACR_LOGIN_SERVER/actions-runner:2.320.0
   ```

3. **Push the Images to ACR**:

   ```powershell
   docker push $ACR_LOGIN_SERVER/gha-runner-scale-set-controller:0.9.3
   docker push $ACR_LOGIN_SERVER/actions-runner:2.320.0
   ```

### Step 5: Clone, Package, and Push the Helm Chart

1. **Clone the GitHub Actions Runner Controller Repository**:

   ```powershell
   git clone https://github.com/actions/actions-runner-controller.git
   cd actions-runner-controller/charts
   ```

2. **Package the Helm Chart**:

   Use the `helm package` command to create `.tgz` files for the charts.

   ```powershell
   cd .\gha-runner-scale-set\
   helm package .
   helm push .\gha-runner-scale-set-0.9.3.tgz oci://hcats0503022020.azurecr.io/helm

   cd ..\gha-runner-scale-set-controller\
   helm package .
   helm push .\gha-runner-scale-set-controller-0.9.3.tgz oci://hcats0503022020.azurecr.io/helm
   ```

### Step 6: Configure `kubectl` to Access the Cluster

Configure `kubectl` to use AKS credentials:

```powershell
az aks get-credentials --resource-group $AKS_RESOURCE_GROUP --name $AKS_CLUSTER_NAME
kubectl get nodes  # Verify connectivity
```

### Step 7: Deploy the ARC Runners Controller from ACR

1. **Create a Namespace for the Controller**:

   ```powershell
   kubectl create namespace $NAMESPACE_ARC_CONTROLLER
   ```

2. **Deploy the Controller** from ACR using Helm:

   ```powershell
   helm install $ARC_CONTROLLER_NAME `
       --namespace $NAMESPACE_ARC_CONTROLLER `
       --version "0.9.3" `
       --set image.repository="$ACR_LOGIN_SERVER/gha-runner-scale-set-controller" `
       oci://$ACR_LOGIN_SERVER/helm/gha-runner-scale-set-controller
   ```

### Step 8: Deploy the ARC Runner Scale Set on AKS

1. **Create a Namespace for the Runners**:

   ```powershell
   kubectl create namespace $NAMESPACE_ARC_RUNNERS
   ```

2. **Deploy the Runner Scale Set using Helm with GitHub PAT**:

   Use your GitHub PAT to authenticate the runner scale set.

   ```powershell
   helm install $ARC_RUNNER_SCALESET_NAME `
       --namespace $NAMESPACE_ARC_RUNNERS `
       --set githubConfigUrl=$GITHUB_CONFIG_URL `
       --set "githubConfigSecret.github_token=$GITHUB_PAT" `
       oci://$ACR_LOGIN_SERVER/helm/gha-runner-scale-set `
       --debug
   ```

### Step 9: Test the Setup with a Workflow

1. **Create a Sample Workflow**: Assuming a sample workflow is provided at `.github/workflows/test-runner.yml`, this workflow should contain a `workflow_dispatch` event to manually trigger it.

2. **Run the Workflow**:
   - Go to the **Actions** tab in your GitHub repository.
   - Select the `test-runner.yml` workflow and click **Run workflow** to manually trigger it.
   - This action should prompt the runner controller to create a pod in AKS to execute the job.

3. **Verify Runner Pod Creation**:

   Check if the runner pod was created and is active in the runner namespace:

   ```powershell
   kubectl get pods -n $NAMESPACE_ARC_RUNNERS --watch
   ```

### Step 10: Debugging Tips

If the workflow remains in a "Waiting for a runner to pick up this job..." state:
- **Check Labels**: Ensure that the `runs-on` label in the workflow matches the runner scale set label (`arc-runner-set` in this example).
- **Inspect Logs**: Check logs for the controller and runner scale set:
  ```powershell
  kubectl logs -n $NAMESPACE_ARC_CONTROLLER -l app.kubernetes.io/name=$ARC_CONTROLLER_NAME
  kubectl logs -n $NAMESPACE_ARC_RUNNERS -l app.kubernetes.io/name=$ARC_RUNNER_SCALESET_NAME
  ```
- **Verify GitHub PAT Permissions**: The GitHub PAT should have `repo`, `workflow`, and `admin:org` permissions.
