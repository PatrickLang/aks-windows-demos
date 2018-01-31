This is my attempt to capture steps needed to deploy:

1. Azure Container Service (AKS) - currently only supporting Linux
2. Virtual-Kubelet - to create a Windows "node"
3. Azure Open Service Broker - make it easy to create Azure SQL databases and other services
4. Helm to deploy a Windows app with web app in container, backend in Azure SQL


```bash
# Change these to be unique
export RGNAME=plang-aks1rg
export AKSNAME=plagn-aks1


```