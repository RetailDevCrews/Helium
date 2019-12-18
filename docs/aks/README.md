# Demo and Deployment Walk Through

## Background

Project Helium is a reusable Advocated Pattern (AdPat). The focus was originally Azure App Services (Web Apps for Containers). The goal is to have a best practices implementation of a C# (TypeScript/Node & Java Springboot are under development) + CosmosDB + Key Vault + Azure Monitor application that project teams can use to build applications that support security, HA, DR and Business Continuity.

### Azure Components in Use

- Azure Container Registry
- Azure Kubernetes Service
  - Linkerd ServiceMesh
  - Prometheus
  - Azure Application Gateway Ingress Controller
  - Azure AAD Pod Identity
- Azure Key Vault
- Azure CosmosDB
- Application Insights

## Demo Install

### Prerequisites

- Azure subscription with permissions to create:
  - Resource Groups, Service Principals, Keyvault, CosmosDB, App Service, Azure Container Registry, Azure Monitor
- Bash shell (tested on Mac, Ubuntu, Windows with WSL2)
  - Will not work in Cloud Shell unless you have a remote dockerd
- Azure CLI 2.0.72+ ([download](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest))
- Docker CLI ([download](https://docs.docker.com/install/))
- .NET Core SDK 2.2 ([download](https://dotnet.microsoft.com/download))
- Visual Studio Code (optional) ([download](https://code.visualstudio.com/download))
- kubectl (install by using `sudo az aks install-cli`)
- Helm v3 ([Install Instructions](https://helm.sh/docs/intro/install/))

### Setup

Fork this repo and clone to your local machine

```shell
cd $HOME

mkdir demo

cd demo

git clone https://github.com/retaildevcrews/helium
```

Change into the base directory of the repo

```shell
cd helium

export REPO_ROOT=$(pwd)
```

Login to Azure and select subscription

```bash

az login

# show your Azure accounts
az account list -o table

# select the Azure account
az account set -s {subscription name or Id}

```

This demo will create resource groups, a CosmosDB instance, Key Vault, Azure Container Registry, and Azure App Service.

Choose a unique name for DNS and resource name prefix.

```bash

# this will be the prefix for all resources
# do not include punctuation - only use a-z and 0-9
# must be at least 5 characters long
# must start with a-z (only lowercase)
export He_Name="youruniquename"

### if true, change He_Name
az cosmosdb check-name-exists -n ${He_Name}

### if nslookup doesn't fail to resolve, change He_Name
nslookup ${He_Name}.vault.azure.net
nslookup ${He_Name}.azurecr.io

```

Create Resource Groups

- When experimenting with this sample, you should create new resource groups to avoid accidentally deleting resources

  - If you use an existing resource group, please make sure to apply resource locks to avoid accidentally deleting resources

- You will create 3 resource groups
  - One for CosmosDB
  - One for ACR
  - One for App Service, Key Vault and Azure Monitor

```bash

# set location
export He_Location=centralus

# resource group names
export He_ACR_RG=${He_Name}-rg-acr
export He_App_RG=${He_Name}-rg-app
export He_Cosmos_RG=${He_Name}-rg-cosmos

# create the resource groups
az group create -n $He_App_RG -l $He_Location
az group create -n $He_ACR_RG -l $He_Location
az group create -n $He_Cosmos_RG -l $He_Location

```

Save your environment variables for ease of reuse and picking up where you left off.

```bash

# run the saveenv.sh script at any time to save He_* variables to ~/${He_Name}.env
# make sure you are in the root of the repo
cd $REPO_ROOT
./saveenv.sh

# at any point if your terminal environment gets cleared, you can source the file
# you only need to remember the name of the env file (or set the $He_Name variable again)
source ~/{yoursameuniquename}.env

```

Create and load sample data into CosmosDB

- This takes several minutes to run
- This sample is designed to use a simple dataset from IMDb of 100 movies and their associated actors and genres
  - See full explanation of data model design decisions [here:](https://github.com/4-co/imdb)

```bash

# set the other Cosmos environment variables
export He_Cosmos_URL=https://${He_Name}.documents.azure.com:443/
export He_Cosmos_DB=imdb
export He_Cosmos_Col=movies

# create the CosmosDB server
az cosmosdb create -g $He_Cosmos_RG -n $He_Name

# create the database
az cosmosdb database create -d $He_Cosmos_DB -g $He_Cosmos_RG -n $He_Name

# create the collection
# 400 is the minimum RUs
# /partitionKey is the partition key
# partition key is the id mod 10
az cosmosdb collection create --throughput 400 --partition-key-path /partitionKey -g $He_Cosmos_RG -n $He_Name -d $He_Cosmos_DB -c $He_Cosmos_Col

# get Cosmos readonly key (used by App Service)
export He_Cosmos_RO_Key=$(az cosmosdb keys list -n $He_Name -g $He_Cosmos_RG --query primaryReadonlyMasterKey -o tsv)

# get readwrite key (used by the imdb import)
export He_Cosmos_RW_Key=$(az cosmosdb keys list -n $He_Name -g $He_Cosmos_RG --query primaryMasterKey -o tsv)

# run the IMDb Import
docker run -it --rm retaildevcrew/imdb-import $He_Name $He_Cosmos_RW_Key $He_Cosmos_DB $He_Cosmos_Col

# Optional: Run ./saveenv.sh to save latest variables

```

Create Azure Key Vault

- All secrets are stored in Azure Key Vault for security
  - This sample uses Managed Identity to access Key Vault

```bash

## create the Key Vault and add secrets
az keyvault create -g $He_App_RG -n $He_Name

# add CosmosDB keys
az keyvault secret set -o table --vault-name $He_Name --name "CosmosUrl" --value $He_Cosmos_URL
az keyvault secret set -o table --vault-name $He_Name --name "CosmosKey" --value $He_Cosmos_RO_Key
az keyvault secret set -o table --vault-name $He_Name --name "CosmosDatabase" --value $He_Cosmos_DB
az keyvault secret set -o table --vault-name $He_Name --name "CosmosCollection" --value $He_Cosmos_Col

```

Setup Container Registry

- Create the Container Registry with admin access _disabled_

```bash

# create the ACR
az acr create --sku Standard --admin-enabled false -g $He_ACR_RG -n $He_Name
```

Create Azure Monitor

- The Application Insights extension is in preview and needs to be added to the CLI

```bash

# Add App Insights extension
az extension add -n application-insights

# Create App Insights
export He_AppInsights_Key=$(az monitor app-insights component create -g $He_App_RG -l $He_Location -a $He_Name --query instrumentationKey -o tsv)

# add App Insights Key to Key Vault
az keyvault secret set -o table --vault-name $He_Name --name "AppInsightsKey" --value $He_AppInsights_Key

# Optional: Run ./saveenv.sh to save latest variables

```

Create your AKS Cluster

Set local variables to use in AKS deployment

```shell
export He_AKS_Name="${He_Name}-aks"
```

Determine the latest version of Kubernetes supported by AKS. It is recommended to choose the latest version not in preview for production purposes, otherwise choose the latest in the list.

```shell
az aks get-versions -l $He_Location -o table

export He_K8S_VER=1.15.5
```

```shell
az aks create --name $He_AKS_Name --resource-group $He_App_RG --location $He_Location --enable-cluster-autoscaler --min-count 3 --max-count 6 --node-count 3 --kubernetes-version $He_K8S_VER --attach-acr $He_Name  --no-ssh-key

az aks get-credentials -n $He_AKS_Name -g $He_App_RG

kubectl get nodes
```

Install AAD Pod Identity for the application

Change directories to the `docs\aks` folder and make the `aad-podid.sh` script executable. Runnig this shell script will deploy AAD Pod Identity to your cluster and assign a Managed Identity.

```shell
cd $REPO_ROOT/docs/aks
sudo chmod +x aad-podid.sh

./aad-podid.sh -a ${He_AKS_Name} -r ${He_App_RG} -m ${He_Name}-msi
```

The output will give the command to provide the MSI with the proper rights to the keyvault created earlier. Run the command as it is shown exactly in the output, (i.e.):

### example

```shell
az keyvault set-policy -n ejvtst1216 --object-id ff4634eb-662f-4faa-9c44-b03e41a09522 --secret-permissions get list --key-permissions get list --certificate-permissions get list
```

The last line of the output will explain the proper label annotation needed when deploying the application. This will be needed later during the application install

```shell
export LABEL="<output from aad-podid.sh>"
```

## Install Helm 3

Install the latest version of Helm by download the latest [release](https://github.com/helm/helm/releases):

```shell
# mac os
OS=darwin-amd64 && \
REL=v3.0.1 && \ #Should be lastest release from https://github.com/helm/helm/releases
mkdir -p $HOME/.helm/bin && \
curl -sSL "https://get.helm.sh/helm-${REL}-${OS}.tar.gz" | tar xvz && \
chmod +x ${OS}/helm && mv ${OS}/helm $HOME/.helm/bin/helm
rm -R ${OS}
```

or

```shell
# Linux/WSL
OS=linux-amd64 && \
REL=v3.0.1 && \ #Should be lastest release from https://github.com/helm/helm/releases
mkdir -p $HOME/.helm/bin && \
curl -sSL "https://get.helm.sh/helm-${REL}-${OS}.tar.gz" | tar xvz && \
chmod +x ${OS}/helm && mv ${OS}/helm $HOME/.helm/bin/helm
rm -R ${OS}
```

Add the helm binary to your path and set Helm home:

```shell
export PATH=$PATH:$HOME/.helm/bin
export HELM_HOME=$HOME/.helm
```

>NOTE: This will only set the helm command during the existing terminal session. Copy the 2 lines above to your bash or zsh profile so that the helm command can be run any time.

Verify the installation with:

```shell
helm version
```

Add the required helm repositories

```shell
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm repo add aad-pod-identity https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts
helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update
```

## Install Linkerd Service Mesh into the cluster

Download the Linkerd v2 CLI:

```shell
# macOS and linux/WSL

curl -sL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin
```

Install the Linkerd control plane in the linkerd namespace:

```shell
linkerd install | kubectl apply -f -
```

Validate the install with:

```shell
linkerd check
```

## Install the NGNIX ingress controller

Create a namespace for your ingress resources. There is a yaml file located in the clones repository under `$REPO_ROOT/docs/aks/cluster/manifests/ingress-nginx-namespace.yaml`

```shell
kubectl apply -f $REPO_ROOT/docs/aks/cluster/manifests/ingress-nginx-namespace.yaml
```

Use Helm to deploy an NGINX ingress controller

```shell
helm install ingress stable/nginx-ingress \
    --namespace ingress-nginx \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
```

Get the Public IP of your Ingress Controller. This will be used later in deploying the application.

```shell
kubectl get svc -n ingress-nginx
```

## Deploy the needed componenets of helium, key rotator and the testing harness

An helm chart is included for the reference application ([helium](https://github.com/RetailDevCrews/helium-csharp))

Install the Helm Chart located in the cloned directory

```shell
cd $REPO_ROOT/docs/aks/cluster/charts
```

Create a file called helm-config.yaml with the following contents that should be edited to fit the environment being deployed in

```yaml
# Default values for helium.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
labels:
  aadpodidbinding: podid #should be value of $LABEL from the output o aad-podid.sh

image:
  repository: retaildevcrew #The spceific acr created for this environment
  name: helium-csharp # The name of the image for the helium-csharp repo

annotations:
  linkerd.io/inject: enabled # Allows for the application to be injected into the Service Mesh

ingress:
  hosts:
    - host: <PUBLICIP_of_INGRESS>.nip.io # Replace the IP address with the IP of the nginx external IP. kubectl get svc -n ingress-nginx to see the correct IP
      paths: /

keyVaultName: helium-aks-kv # Replace with the name of the Key Vault that holds the secrets
```

This file can now be given to the the helm install as an override to the default values.

```shell
helm install helium-aks helium --set image.repository=<acr_name>.azurecr.io -f helm-config.yaml
```

## Dashboard setup

Open `AKS Dashboard.json` in an editor and update `{SUBSCRIPTION_GUID}` with your subscription, `{AKS_RESOURCE_GROUP}` with your AKS cluster resource group name, and `{COSMOS_RESOURCE_GROUP}` with your Cosmos resource group name.

Navigate to ([Dashboard](https://portal.azure.com/#dashboard)) within your Azure portal. Click upload and select the `AKS Dashboard.json` file with your correct subscription GUID, resource group names, and app name.

For more documentation on creating and sharing Dashboards, see ([here](https://docs.microsoft.com/en-us/azure/azure-portal/azure-portal-dashboards)).

## Optional

A testing application was written to stress test the application and drive the Request Units on the CsomsoDB. You can deploy the application to AKS as a cronjob. The cronjobs can be deployed to your cluster via a helm chart located at `REPO_ROOT/docs/aks/cluster/charts`.

```shell
cd $REPO_ROOT/docs/aks/cluster/charts

helm install helium-smoker smoker --set ingressURL=http://<IP_OF_INGRESS>.nip.io

#Verify the CronJobs are in the cluster
kubectl get cronjobs
```

The cronjobs are set to run for 7.5 minutes every 20 minutes.
