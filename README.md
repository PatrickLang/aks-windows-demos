This is my attempt to capture steps needed to deploy:

1. Azure Container Service (AKS) - currently only supporting Linux
2. Virtual-Kubelet - to create a Windows "node"
3. Azure Open Service Broker - make it easy to create Azure SQL databases and other services
4. Helm to deploy a Windows app with web app in container, backend in Azure SQL


Prerequisites

```bash
# login
az login 
az account set --subscription <redacted>

# Opt into AKS preview
az provider register -n Microsoft.ContainerService
```


Now deploy the AKS cluster

```bash
# Change these to be unique
export RGNAME=plang-aks1rg
export AKSNAME=plang-aks1

# Change this if you want - pick from 'eastus,westeurope,centralus,canadacentral,canadaeast'
export AKSREGION=eastus

az group create --location=$AKSREGION --name=$RGNAME
az aks create --resource-group $RGNAME --name=$AKSNAME --node-count 1 --generate-ssh-keys


# Now get the credentials to connect
az aks get-credentials --resource-group $RGNAME --name $AKSNAME
```

If you've been playing with Kubernetes on Azure, you'll probably have multiple configs on your machine. To keep track of them, use `kubectl config get-contexts`. The last one you downloaded with `az aks get-credentials` is selected, but you can always switch back to the others with `kubectl config use-context`