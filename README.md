# Bastion-Workloadvm-privatelink
## Overview

1) Create a central proxy scaleset in the bastion vnet to monitor and control web traffic from the workload vnet. 
2) Create a Jump Server to access a scaleset in the worklod vnet. 
3) Create a scaleset in the workload vnet. 
4) Create a private endpoint in the bastion vnet to connect to the private link service of the workload vnet. 
5) Create a private endpoint in the workload vnet to connect to the private link service of the bastion vnet.  

## Pre-requisites 
The Azure CLI version is 2.6. Use the following command to install on Linux
```powershell
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash 
```
### Architecture Diagram
* Process flow ![alt text](https://github.com/preddy727/Bastion-Workloadvm-privatelink/blob/master/architecture.png)

## Goals of the Lab
1. Create all the components described in the overview and architecture diagram.    

### Create the Bastion components (VNET, Subnets, Proxy ScaleSet, Jump server, External Load Balancer, Private Link Service) 
```powershell 
## Create a resource group 
az group create --name Bastion --location eastus2

##Create a virtual network 
az network vnet create --resource-group Bastion --name myVirtualNetwork --address-prefix 10.0.0.0/16

##Create a subnet for the squid proxy 
az network vnet subnet create --resource-group Bastion --vnet-name myVirtualNetwork --name mySubnet \
    --address-prefixes 10.0.0.0/24

##Create a subnet for the jump server
az network vnet subnet create --resource-group Bastion --vnet-name myVirtualNetwork --name myJumpSubnet \
    --address-prefixes 10.0.1.0/24

##Create a zone-redundant scale set installed with squid and attached to external load balancer. 
##The sample cloudinit is in this repository. 
az vmss create \
    --resource-group Bastion \
    --name myScaleSet \
    --image UbuntuLTS \
    --upgrade-policy-mode automatic \
    --admin-username azureuser \
    --generate-ssh-keys \
    --zones 1 2 3 \
    --vnet-name myVirtualNetwork \
    --subnet mySubnet \
    --custom-data MyCloudInitScript.yml
    
##Disable Private Link service network policies on subnet
az network vnet subnet update --resource-group Bastion --vnet-name myVirtualNetwork --name mySubnet \
    --disable-private-link-service-network-policies true

##Create a Private Link Service 
az network private-link-service create \
--resource-group Bastion \
--name myPLS \
--vnet-name myVirtualNetwork \
--subnet mySubnet \
--lb-name myScaleSetLB \
--lb-frontend-ip-configs loadBalancerFrontEnd \
--location eastus2 

##Note the resource id which will be needed later when creating an endpoint in the workload VNET 
"/subscriptions/<your sub id>/resourceGroups/Bastion/providers/Microsoft.Network/privateLinkServices/myPLS"

##Create a Jump Server

az vm create --image UbuntuLTS --generate-ssh-keys --admin-username 4soadmin --location eastus2 --name 4solinuxvm --resource-group Bastion --size Standard_D3_v2 --vnet-name myVirtualNetwork --subnet myJumpSubnet --nsg "" --output table

##Validate connectivity to Jump Server
just replace the public IP with your specific IP: ssh 4soadmin@<publicIP>



```

### Create the workload components 
```powershell
## Create a resource group 
az group create --name Workload --location eastus2
## Create the virtual network 
az network vnet create \
--resource-group Workload \
--name myPEVnet \
--address-prefix 10.0.0.0/16

## Create the subnet 
az network vnet subnet create \
--resource-group Workload \
--vnet-name myPEVnet \
--name myPESubnet \
--address-prefixes 10.0.0.0/24

## Disable private endpoint network policies on subnet 
az network vnet subnet update \
--resource-group Workload \
--vnet-name myPEVnet \
--name myPESubnet \
--disable-private-endpoint-network-policies true

##Create private endpoint and connect to private link service 
az network private-endpoint create \
--resource-group Workload \
--name myPE \
--vnet-name myPEVnet \
--subnet myPESubnet \
--private-connection-resource-id \
"/subscriptions/<your sub id>/resourceGroups/Bastion/providers/Microsoft.Network/privateLinkServices/myPLS" \
--connection-name myPEConnectingPLS \
--location eastus2

az network private-link-service show --resource-group Bastion --name myPLS

##Create a scaleset and use the proxy endpoint. Go to the portal and search for myPE. Record the private ip address. Update the myclientcloudinit.yml with the private ip for the proxy settings. 

az vmss create \
    --resource-group Bastion \
    --name myScaleSet \
    --image UbuntuLTS \
    --upgrade-policy-mode automatic \
    --admin-username azureuser \
    --generate-ssh-keys \
    --zones 1 2 3 \
    --vnet-name myPEVnet \
    --subnet myPESubnet \
    --custom-data myclientcloudinit.yml


```
### Create two security groups. Allow ssh from on premises and incoming traffic from workload vnet 

### Setup SSH access from Bastion Jump server

### Verify access to external urls

