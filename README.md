This is my attempt to capture steps needed to deploy:

1. Azure Container Service (AKS) - currently only supporting Linux
2. Virtual-Kubelet - to create a Windows "node"
3. Azure Open Service Broker - make it easy to create Azure SQL databases and other services
4. Helm to deploy a Windows app with web app in container, backend in Azure SQL


## Prerequisites

I do all my management with Ubuntu under the [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/about). Once that's set up, you need to install:

- [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest)
- [Helm](https://github.com/kubernetes/helm/blob/master/docs/install.md)


```bash
# login
az login 
az account set --subscription <redacted>

# Opt into AKS preview
az provider register -n Microsoft.ContainerService
```


## Deploy the AKS cluster

This is mostly lifted from Azure's [Kubernetes Walkthrough](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough)

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


## Deploy a Linux app

`kubectl apply -f azure-vote.yml`

Now - wait on the public service IP with

`kubectl get svc -w` 

and watch for azure-vote-front to have an IP. Then connect to it from a browser

Once you're done, delete it with 

```bash
kubectl delete deploy azure-vote-front
kubectl delete deploy azure-vote-back
kubectl delete svc azure-vote-front
kubectl delete svc azure-vote-back
```


### Optional - Upgrade it

You can upgrade to a new version easily with two commands. Find the version you want with `az aks get-versions`, then upgrade with `az aks upgrade --kubernetes-version <version> --resource-group <rgname> --name <aks name>` using the name and resource group name from above.

## Deploy the Virtual Kubelet

First, make sure Helm is initialized. Run `kubectl get pods --namespace kube-system` - if you don't see **tiller** listed, then run `helm init` before moving on.


> To find a good region, see Azure Container Instances [Region Availability](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-quotas) for Windows.  As of 3/2, Windows is in WestEurope, WestUS, EastUS, SoutheastAsia


This assumes you still have RGNAME and AKSNAME set from earlier.

```bash
az aks install-connector --resource-group $RGNAME --name=$AKSNAME --os-type Windows --connector-name vk1
```

That will drop a helpful hint to check that it's running. It will be something like this

```
NOTES:
The virtual kubelet is getting deployed on your cluster.

To verify that virtual kubelet has started, run:

  kubectl --namespace=default get pods -l "app=vk1-windows-virtual-kubelet"
```

Once that pod is up, `kubectl get node` will show the virtual Windows node running:

```
$ kubectl get node
NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-27183287-0   Ready     agent     28m       v1.7.7
virtual-kubelet-vk1-win    Ready     agent     1m        v1.8.3
```


### temporary workaround - missing SP

> example log & github issue link needed

```
kubectl log <podname>


# Get service principal GUID
cat .azure/aksServicePrincipal.json
az role assignment create --assignee "..." --resource-group plang-aks1 --role Contributor
```


### Deploy the Windows app 

> Note: Stuff from this point isn't working yet


`kubectl apply -f whoami.json`


> Note: For apps to run under ACI, they need to have the right tolerations to allow scheduling on aci, and node selectors to pick the windows node. See whoami.json for an example.



After some time, `kubectl get pod` should show the pod as _Running_. If it doesn't, check the logs from the virtual kubelet.

```bash
# Get pod name - same as checking the status after running az aks install-connector
kubectl --namespace=default get pods -l "app=vk1-windows-virtual-kubelet-for-aks"

# Check logs
kubectl logs vk1-windows-virtual-kubelet-for-aks-1491062091-7qdb7
```

You should get a helpful hint like this:

```
2018/03/02 23:34:09 Error creating pod 'boisterous-greyhound-iis-static-84fb585686-brj7x': api call to https://management.azure.com/subscriptions/<redacted subid>/resourceGroups/plang-aks1rg/providers/Microsoft.ContainerInstance/containerGroups/default-boisterous-greyhound-iis-static-84fb585686-brj7x?api-version=2017-12-01-preview: got HTTP response status code 400 error code "ResourceSomeRequestsNotSpecified": The 'MemoryInGB' request is not specified in 'Microsoft.Azure.CloudConsole.Providers.Common.Entities.ComputeResources' in container 'iis-static' of contain group 'default-boisterous-greyhound-iis-static-84fb585686-brj7x'. It is required since API version '2017-07-01-preview'.
```


## Deploy a Windows app with Helm


```
helm create iis-static
```


Check the status with `helm list`

After you modify your chart or values, then `helm upgrade <name> <chart directory>`


Here's what it looks like once the Helm chart is installed and pod is running:

```
patrick@plang-h1:~/6385b8a593a3519f81bb096c9edda6b2$ helm list
NAME                    REVISION        UPDATED                         STATUS          CHART                           NAMESPACE
boisterous-greyhound    5               Fri Mar  2 15:51:52 2018        DEPLOYED        iis-static-0.1.0                default
vk1-windows             1               Fri Mar  2 15:10:55 2018        DEPLOYED        virtual-kubelet-for-aks-0.1.3   default
patrick@plang-h1:~/6385b8a593a3519f81bb096c9edda6b2$ kubectl get pod
NAME                                                   READY     STATUS    RESTARTS   AGE
boisterous-greyhound-iis-static-76c9c8bcc4-4bj9n       1/1       Running   0          8m
vk1-windows-virtual-kubelet-for-aks-1491062091-7qdb7   1/1       Running   0          24m
```


### Mistakes I made

At first, I didn't set a container memory limit. I found this error from the virtual kubelet:

```
2018/03/02 23:34:09 Error creating pod 'boisterous-greyhound-iis-static-84fb585686-brj7x': api call to https://management.azure.com/subscriptions/<redacted subid>/resourceGroups/plang-aks1rg/providers/Microsoft.ContainerInstance/containerGroups/default-boisterous-greyhound-iis-static-84fb585686-brj7x?api-version=2017-12-01-preview: got HTTP response status code 400 error code "ResourceSomeRequestsNotSpecified": The 'MemoryInGB' request is not specified in 'Microsoft.Azure.CloudConsole.Providers.Common.Entities.ComputeResources' in container 'iis-static' of contain group 'default-boisterous-greyhound-iis-static-84fb585686-brj7x'. It is required since API version '2017-07-01-preview'.
```

My fix was to add this to iis-static/values.yaml:

```
resources:
  limits:
    memory: 1G 
```


Which led to the similar errors:

```
The 'Cpu' request is not specified in 'Microsoft.Azure.CloudConsole.Pro
viders.Common.Entities.ComputeResources' in container 'iis-static' of contain group 'default-boisterous-greyhound-iis-static-6c65cc687-hxswb'. It is required since API
version '2017-07-01-preview'.

...

The 'Cpu' request is not specified in 'Microsoft.Azure.CloudConsole.Pro
viders.Common.Entities.ComputeResources' in container 'iis-static' of contain group 'default-boisterous-greyhound-iis-static-6c65cc687-snhwd'. It is required since API
version '2017-07-01-preview'.
```

So now values.yaml:

```
resources:
  limits:
    memory: 1G 
    cpu: 1
  requests:
    cpu: 1
 
```


