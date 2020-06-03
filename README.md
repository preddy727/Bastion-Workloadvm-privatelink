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



##Create a Jump Server

az vm create --image UbuntuLTS --generate-ssh-keys --admin-username 4soadmin --location eastus2 --name 4solinuxvm --resource-group Bastion --size Standard_D3_v2 --vnet-name myVirtualNetwork --subnet myJumpSubnet --nsg "" --output table

##Validate connectivity to Jump Server
just replace the public IP with your specific IP: ssh 4soadmin@<publicIP>

##Enable the Azure AD (AAD) Login extension for Linux to this vm 

az vm extension set --publisher Microsoft.Azure.ActiveDirectory.LinuxSSH --name AADLoginForLinux --resource-group Bastion --vm-name 4solinuxvm

##Assign the Virtual Machine Administrator login to your current Azure user

vm=$(az vm show --resource-group Bastion --name 4solinuxvm --query id -o tsv)

az role assignment create \
    --role "Virtual Machine Administrator Login" \
    --assignee $username \
    --scope $vm

##Validate connectivity using ssh and AAD user. Authenticate using multi-factor authentication. Enter code in browser. 
ssh <yourAadUser@domain.com>@<publicIP>
##Install the azure cli on the jump server
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

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

##Look up the ip of the Scaleset instances in the portal 
ssh azureuser@instanceip 

Configuring Squid Proxy Server
The Squid configuration file is found at /etc/squid/squid.conf.

1. Open this file in your text editor with the command:

sudo nano /etc/squid/squid.conf
2. Navigate to find the http_port option. Typically, this is set to listen on Port 3218. This port usually carries TCP traffic. If your system is configured for traffic on another port, change it here.

You may also set the proxy mode to transparent if you’d like to prevent Squid from modifying your requests and responses.

Change it as follows:

http_port 1234 transparent
3. Navigate to the http_access deny all option. This is currently configured to block all HTTP traffic. This means no web traffic is allowed.

Change this to the following:

http_access allow all
4. Navigate to the visible_hostname option. Add any name you’d like to this entry. This is how the server will appear to anyone trying to connect. Save the changes and exit.

5. Restart the Squid service by entering:

sudo systemctl restart squid


Block Websites on Squid Proxy
1. Create and edit a new text file /etc/squid/blocked.acl by entering:

sudo nano /etc/squid/blocked.acl
2. In this file, add the websites to be blocked, starting with a dot:

.facebook.com

.twitter.com

Note: The dot specifies to block all subsites of the main site.

3. Open the /etc/squid/squid.conf file again:

sudo nano /etc/squid/squid.conf
4. Add the following lines just above your ACL list:

acl blocked_websites dstdomain “/etc/squid/blocked.acl”
http_access deny blocked_websites

```

### Create the workload components 
```powershell
## Create a resource group 
az group create --name Workload --location eastus2

##Create a scaleset with an internal load balancer and use the proxy endpoint. Go to the portal and search for myPE. Record the private ip address. Update the myclientcloudinit.yml with the private ip for the proxy settings. 

az group deployment create --resource-group workload \
--template-uri https://github.com/preddy727/Bastion-Workloadvm-privatelink/blob/master/template.json 
Please provide string value for 'vmssName' (? for help): workload
Please provide int value for 'instanceCount' (? for help): 2
Please provide string value for 'adminUsername' (? for help): prreddy
Please provide securestring value for 'adminPasswordOrKey' (? for help):


## Disable private endpoint network policies on subnet 
az network vnet subnet update \
--resource-group Workload \
--vnet-name workloadwvnet \
--name workloadwsubnet \
--disable-private-endpoint-network-policies true

##Create private endpoint and connect to private link service 
az network private-endpoint create \
--resource-group Workload \
--name myPE \
--vnet-name workloadwvnet \
--subnet workloadwsubnet \
--private-connection-resource-id \
"/subscriptions/<your sub id>/resourceGroups/Bastion/providers/Microsoft.Network/privateLinkServices/myPLS" \
--connection-name myPEConnectingPLS \
--location eastus2

az network private-link-service show --resource-group Bastion --name myPLS




```
### Create two security groups. Allow ssh from on premises and incoming traffic from workload vnet 

### Setup SSH access from Bastion Jump server

### Verify access to external urls

