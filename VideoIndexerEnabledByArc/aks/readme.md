# Video Indexer Arc Extension

## About

Video Indexer Arc Enabled Solution is an Azure Arc Extension Enabled Service aimed at running Video and Audio Analysis on Edge Devices. The solution is designed to run on Azure Arc-enabled Kubernetes and supports many video formats, including MP4 and four additional common formats. The solution supports many languages in all basic audio-related models and assumes that one VI resource is mapped to one extension.

The purpose of this document is to present the onboarding steps and pre-requisites required for Cluster Administrator, IT Operator, Dev Ops and Engineering teams to enable Video Indexer as arc extension on their current Infrastructure.

## Prerequisites

> **_Note_:** In order to successfully deploy the VI Extension, you **must first** submit your subscription for approval and be approved. Please use this application link. [this form](https://aka.ms/vi-register).

- Azure subscription with permissions to create Azure resources
- Azure Video Indexer Account. The quickest way is using the Azure Portal using this tutorial [Create Video Indexer account](https://learn.microsoft.com/azure/azure-video-indexer/create-account-portal#use-the-azure-portal-to-create-an-azure-video-indexer-account).
- The latest version of [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli). You can skip if you're using cloud shell.
- The latest version of connectedk8s Azure CLI extension, installed by running the following command. **You can skip if you're using the Cloud Shell** option:

```bash
az extension add --name connectedk8s
az provider register -n 'Microsoft.Kubernetes' 
az provider register -n 'Microsoft.KubernetesConfiguration' 
az provider register -n 'Microsoft.ExtendedLocation'
```

## Parameters

| Parameter | Type | Description |
|-----------|---------|-------------|
| accountResourceId |  string | The Video Indexer Full Resource Id that will be connected to the Arc Extension
| accountId  | string | The Video Indexer Account Id
| videoIndexerEndpointUri | string | Video Indexer Dns Name to be used as the Portal endpoint
| arcConnectedClusterName | string | The Azure Arc Kubernetes Cluster Created at Step 1
| identityId  | string | a User Assigned Managed Identity with 'Contributor' Role Assignment on your subscription

The Video Indexer API is a Kubernetes service that needs to be exposed in order to be accessible from outside the cluster. This can be achieved by configuring an appropriate external address (DNS/IP) to route incoming traffic to the cluster.
Here are the common methods for exposing the service:

- NodePort:
Use the IP address of the node and a NodePort. This approach exposes the service on a specific port on all nodes in the cluster. External traffic can access the service by using the node's IP address and the assigned NodePort.
- LoadBalancer:
In environments that support external load balancers (such as MetalLB for OnPrem clusters), you can use the LoadBalancer service type. This will provision an external IP address that routes traffic to your service.
- Ingress:
Use an Ingress resource in conjunction with an Ingress controller. This method provides a single point of access to multiple services within the cluster, typically using a DNS name. The Ingress controller can handle SSL termination, load balancing, and path-based routing.
- Dedicated Pod IP (Relay Traffic):
If relay traffic is enabled, a dedicated Pod IP can be used to route traffic. This method is less common and typically used in specialized network configurations.

## 1. One-Click Deploy Sample to Azure
**This step is optional.** If you would like to test Video Indexer Edge Extension on a sample edge device, this deployment script can be used to quickly set up a K8S cluster and all pods to run VI on Edge. This script will deploy the following resources:
- Small 2 node AKS Cluster (costs are ~$0.80/hour)
- Enable ARC Extension on top of the cluster
- Add Video Indexer Arc Extension
- Expose the Video Indexer Swagger API for dataplane operations

You can read more on how to set up your cloud shell environment [here](https://learn.microsoft.com/azure/cloud-shell/quickstart?tabs=azurecli).

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://shell.azure.com/bash?url=)

In the cloud shell execute the following command:

```bash
wget -SSL https://raw.githubusercontent.com/Azure-Samples/media-services-video-indexer/master/VideoIndexerEnabledByArc/aks/vi_extension_install.sh

chmod +x ./vi_extension_install.sh

sh vi_extension_install.sh
```

> **_Note_:** The script aim to run on Ubuntu OS and contains command that uses Ubuntu package manager. 

During the deployment the script will ask the following questions where you will need to provide your environment specific values. Below table explains each question and the desired value. Some will expect or have default values.

| Question | value | Details
| --- | --- | --- |
| What is the Video Indexer account ID during deployment? | GUID | Your Video Indexer Account ID |
| What is the Azure subscription ID during deployment? | GUID | Your Azure Subscription ID |
| What is the name of the Video Indexer resource group during deployment? | string | The Resource Group Name of your Video Indexer Account |
| What is the name of the Video Indexer account during deployment? | string | Your Video Indexer Account name |


Once deployed you will get a URL to the Data Plane API of your new Video Indexer on Edge extension which is now running on the AKS cluster. You can use this API to perform indexing jobs and test Video Indexer on Edge. Please note that this is **not** meant as a path to production and only provided to quickly test Video Indexer on Edge functionality. This concludes the demo script and you are done. Below are the steps if you want to deploy VI on Edge manually.

## 2. Manual deployment steps start here

Follow these steps to deploy the Video Indexer Arc Extention to your Arc K8S Enabled cluster. 

### Hardware and Software Requirements

### Minumum Hardware Requirements

The following are the minimum and recommended requirements respectively.

| Configuration | VM Count | Node CPU Cores Count*  | Node Ram | Node Storage | Remarks
| --- | --- | --- | --- | --- | --- |
| Minimum | 1 | 32 Cores | 64 GB | 50 GB | Process **on** video in **basic audio** preset
| Recommended | 3 | 48-64 Cores | 256 GB | 100 GB | Processing **10 videos** in parallel in **basic video** preset

*Storage needs to support **ReadWriteMany** Storage Class  
**VM CPU must support [AVX2](https://en.m.wikipedia.org/wiki/Advanced_Vector_Extensions) extensions**  

> **_Note_:** at least 2-node cluster is recommended for high availability and scalability.

> **_Tip_:** We recommend creating a dedicate node-pool / auto-scaling groups to host the VI Solution


### Minimum Software Requirements

| Component |  Minimum Requirements
| --- | ---
| Operating System | Ubuntu 22.04 LTS or any Linux Compatible OS
| Kubernetes | > 1.29
| Azure CLI | > 2.64.0

## Installation Steps

### Step 1 - Create Azure Arc Kubernetes Cluster and connect it to your cluster

> **_Note_:**
> The following command assumes you have a kubernetes cluster and that the Current Context on your ./kube/config file points to it.

Run the following command to connect your cluster. This command deploys the Azure Arc agents to the cluster and installs Helm v. 3.6.3 to the .azure folder of the deployment machine. This Helm 3 installation is only used for Azure Arc, and it does not remove or change any previously installed versions of Helm on the machine.

```bash
az connectedk8s connect --name myAKSCluster --resource-group myResourceGroup
```

> **_Tip:_** Follow the article [how to connect your cluster to Azure Arc][4] on Azure Docs
> for a complete walkthrough on this process

[4]: https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/quickstart-connect-cluster?tabs=azure-cli


```bash
az rest --method get --uri <the uri from the location response header>
```

If the response is 409 (conflict), it means the resources already exist for your subscription and you can proceed to the below command without waiting.


### Step 2 - Create Azure Arc Video Indexer Extension using CLI

The following parameters will be used as input to the extension creation command:

| Parameter | Default | Description
|-----------|---------|-------------
| release-namespace | default | The kubernetes namespace which the extension will be installed into
| cluster-name | | The kubernetes azure arc instance name
| resource-group | | The kubernetes azure arc resource group name
| version | latest | Video Indexer Extension version
| videoIndexer.accountId |  | Video Indexer Account Id
| videoIndexer.endpointUri |  | Video Indexer Dns Name to be used as the Portal endpoint

```bash
az k8s-extension create --name videoindexer \
    --extension-type Microsoft.videoindexer \
    --scope cluster \
    --release-namespace ${namespace} \
    --cluster-name ${connectedClusterName} \
    --resource-group ${connectedClusterRg} \
    --cluster-type connectedClusters \
    --auto-upgrade-minor-version true \
    --config "videoIndexer.accountId=${viAccountId}" \
    --config "videoIndexer.endpointUri=${dnsName}" 
    --config "storage.storageClass=azurefile-csi" \
    --config "storage.accessMode=ReadWriteMany" \
    --config "ViAi.gpu.enabled=true" \
    --config "ViAi.gpu.tolerations.key=nvidia.com/gpu" \
    --config "ViAi.gpu.nodeSelector.workload=summarization" \
    --config "scaling.ai.maxReplicaCount=${maxAiReplicaCount}

```

There are some additional Parameters that can be used in order to have a fine grain control on the extension creation

| Parameter | Default | Description
|-----------|---------|-------------
| AI.nodeSelector | - | The node Selector label on which the AI Pods (speech and translate)  will be assigned to
resource.requests.mem
| videoIndexer.webapi.resources.requests.cpu | 0.5 | The request number of cores for the web api pod
| videoIndexer.webapi.resources.requests.mem | 4Gi | The request memory capacity for the web api pod
| videoIndexer.webapi.resources.limits.cpu | 1 | The limits number of cores for the web api pod
| videoIndexer.webapi.resources.limits.mem | 6Gi | The limits memory capacity for the web api pod
| videoIndexer.webapi.resources.limits.mem | 6Gi | The limits memory capacity for the web api pod
| storage.storageClass | "" | The storage class to be used
| storage.useExternalPvc | false | determines whether an external PVC is used. if true, the VideoIndexer PVC will not be installed

example deploy script :

```bash
az k8s-extension create --name videoindexer \
    --extension-type Microsoft.videoindexer \
    .......

    --config AI.nodeSelector."beta\\.kubernetes\\.io/os"=linux
    --config "videoIndexer.webapi.resources.requests.mem=4Gi"\
    --config "videoIndexer.webapi.resources.limits.mem=8Gi"\
    --config "videoIndexer.webapi.resources.limits.cpu=1"\
    --config "storage.storageClass=azurefile-csi" 

```

### Step 3 (Alternative) - Deploy Using Bicep Template

In case you do not wish to use Az CLI Commands to deploy the video indexer arc extension , you can use the bicep deployment
provided on the current folder.
> **_Note:_**: This Step replaces the need to run Step 2 + 3 above
> **_Note:_**: In order to deploy the Bicep template you will need to use user-assigned Managed Identity with a 'Contributor' Role Assignment on the Subscription.

1. Open The [Template File](vi.arcextension.template.bicep) file and Fill in the required parameters (see below).
2. Run the Following Az CLi Commands in order to deploy the template using the [az deployment group create](https://learn.microsoft.com/en-us/cli/azure/deployment/group?view=azure-cli-latest#az-deployment-group-create) command.

```shell
az deployment group create --resource-group myResourceGroup --template-file .\vi.arcextension.template.bicep
```

### Step 4 - Verify Deployment

```bash
kubectl get pods -n video-indexer
```

you will see the video indexer pods are up and running.

> **_Note_:** It might take few minutes for all the pods to become available and running .

# How To Acccess the extension :
**_Note_:** Please make sure your end of line sequence is LF and not CRLF for the script to work right away.
```bash
subscription="<your azure subscription>"
rg="<your vi account rg>"
viaccountName="your vi account name"
azToken=$(az account get-access-token --resource https://management.azure.com/ --query accessToken -o tsv)

response=$(az rest --method post \
    --uri "https://management.azure.com/subscriptions/$subscription/resourcegroups/$rg/providers/Microsoft.VideoIndexer/accounts/$viaccountName/generateExtensionAccessToken?api-version=2023-06-02-preview" \
    --headers "accept=application/json" "Authorization=Bearer $azToken" "Content-Type=application/json" \
    --body '{
        "permissionType": "Contributor",
        "scope": "Account",
        "extensionId": "<your azure extension id from the azure portal or by az cli>"
    }')

## Extract the access token from the response
extensionAccessToken=$(echo $response | jq -r '.accessToken')

echo $extensionAccessToken
```
