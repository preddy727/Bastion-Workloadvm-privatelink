# Bastion-Workloadvm-privatelink
## Overview

1) Create a central proxy scaleset in the bastion vnet to monitor and control web traffic from the workload vnet. 
2) Create a Jump Server to access a scaleset in the worklod vnet. 
3) Create a scaleset in the workload vnet. 
4) Create a private endpoint in the bastion vnet to connect to the private link service of the workload vnet. 
5) Create a private endpoint in the workload vnet to connect to the private link service of the bastion vnet.  

## Pre-requisites 
The Azure CLI version is 2.6. 
```powershell
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash 
```
### Architecture Diagram
* Process flow ![alt text](https://github.com/preddy727/Bastion-Workloadvm-privatelink/blob/master/architecture.png)

## Goals of the Lab
1. Create all the components described in the overview and architecture diagram.    

### Create the Bastion VNET
```powershell 
## Create a resource group 
az group create --name Bastion --location eastus2

##Create a virtual network 
az network vnet create --resource-group Bastion --name myVirtualNetwork --address-prefix 10.0.0.0/16

##Create a subnet 
az network vnet subnet create --resource-group Bastion --vnet-name myVirtualNetwork --name mySubnet --address-prefixes 10.0.0.0/24
##Create a Internal load balancer
az network lb create --resource-group Bastion --name myILB --sku standard --vnet-name MyVirtualNetwork --subnet mySubnet --frontend-ip-name myFrontEnd --backend-pool-name myBackEndPool
##Create a load balancer health probe 
az network lb probe create \
    --resource-group Bastion \
    --lb-name myILB \
    --name myHealthProbe \
    --protocol tcp \
    --port 80
##Create a load balancer rule
az network lb rule create \
    --resource-group Bastion \
    --lb-name myILB \
    --name myHTTPRule \
    --protocol tcp \
    --frontend-port 80 \
    --backend-port 80 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe
##Create a zone-redundant scale set 
az vmss create \
    --resource-group Bastion \
    --name myScaleSet \
    --image UbuntuLTS \
    --upgrade-policy-mode automatic \
    --admin-username azureuser \
    --generate-ssh-keys \
    --zones 1 2 3
    --custom-data MyCloudInitScript.yml
```

### Attach an external load balancer

### Create two security groups. Allow ssh from on premises and incoming traffic from workload vnet 

### Create the Workload VNET

### Install the Bastion Squid proxy VM using the Ubuntu marketplace image

### Intall Squid
```powershell 
sudo yum install squid
sudo chkconfig squid on
sudo service squid start
sudo service squid status
```
### Setup private link service in Bastion 

### Setup private link service in Workload 

### Setup SSH access from Bastion Jump server

### Verify access to external urls

