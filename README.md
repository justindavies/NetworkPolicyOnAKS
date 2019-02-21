# Network Policy on AKS
This is a demo to walk through setting up Network Policies on AKS, and their use.  By default, without any Network Policies, any Pods within a cluster namespace are allowed to communicate with each other.  From a security persepctive you can wither trust the boundary of the cluster ot implement hard restrictions on workload to workload communications.

## Why!?
Anything running within a container is still an attack vector.  If you are serving a REST endpoint over HTTP, you can still (potentially) inflict a buffer overflow.  Containers don't change this.  They are just a mechanism to Build, Ship and Run your workloads.  If yout B.S.R. bad code it just means it gets shipped quiccker.

### Now what?
With Network Policies you have the ability to limit communication from one workload to another.  This allows you to limit the attack vector, and constrain communication to ONLY the workloads that need to talk to each other.  "Every little helps"...

### 

### Yes, but I need you to secure everything
Here's a few tips:

* Scan all code and containers with something like Aqua or Twistlock
* Use Mutual TLS for inter-Pod communications
* Setup namespacaes for Dev/Test/Production
* NEVER run workloads as root (superuser)
* Restrict Container processes to what they need to access (disk, memory limits, CPU limits)

## Enable the preview feature

As this is a preview feature, register it on your account:

```bash
az feature register --name EnableNetworkPolicy --namespace Microsoft.ContainerService
```

This process will take a few minutes to propogate to your subscription, to check the status issue a query to the Azure API:

```bash
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/EnableNetworkPolicy')].{Name:name,State:properties.state}"

Name                                            State
----------------------------------------------  -----------
Microsoft.ContainerService/EnableNetworkPolicy  Registering
```

## Creating your AKS cluster with Network Policy support

Let's go ahead and setup some environment variables we will use later...

```bash
export RESOURCE_GROUP_NAME=juda-AK-NP
export CLUSTER_NAME=juda-AKS
```

***Use your Microsoft alias to setup your cluster RG***


### Create a resource group for your new AKS cluster:

```bash
az group create --name $RESOURCE_GROUP_NAME --location eastus
```


### Create a virtual network and subnet
```bash
az network vnet create --resource-group $RESOURCE_GROUP_NAME --name myVnet --address-prefixes 10.0.0.0/8 --subnet-name myAKSSubnet --subnet-prefix 10.240.0.0/16
```


### Get the virtual network resource ID
```bash
VNET_ID=$(az network vnet show --resource-group $RESOURCE_GROUP_NAME --name myVnet --query id -o tsv)
```


### Get the virtual network subnet resource ID
```bash
SUBNET_ID=$(az network vnet subnet show --resource-group $RESOURCE_GROUP_NAME --vnet-name myVnet --name myAKSSubnet --query id -o tsv)
```

### Create the AKS cluster and specify the virtual network and service principal information

```bash
az aks create --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME --kubernetes-version 1.12.4 --network-plugin azure --service-cidr 10.0.0.0/16 --dns-service-ip 10.0.0.10 --docker-bridge-address 172.17.0.1/16 --vnet-subnet-id $SUBNET_ID 
```

***Note: You must specify the Kubernetes version to enable Network Policies as it is version specific where the feature has been lit up*** 

It's time for a :coffee: (or :tea:) - this will take a few minutes.


```bash
DnsPrefix                   EnableRbac    Fqdn                                                      KubernetesVersion    Location    Name      NodeResourceGroup              ProvisioningState    ResourceGroup
--------------------------  ------------  --------------------------------------------------------  -------------------  ----------  --------  -----------------------------  -------------------  ---------------
juda-AKS-juda-AK-NP-63bb10  True          juda-aks-juda-ak-np-63bb10-e3e56c90.hcp.eastus.azmk8s.io  1.12.4               eastus      juda-AKS  MC_juda-AK-NP_juda-AKS_eastus  Succeeded            juda-AK-NP
```

### Get AKS credentials

Once you have an AKS cluster up and running, you need to download the Kubernetes context to be able to connect via **kubectl**:

```bash
az aks get-credentials --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME
```
