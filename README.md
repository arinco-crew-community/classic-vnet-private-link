# Connect a Cloud Service to a Azure SQL DB using Private Link (Preview)

This blog post details how to use az and azure cli to connect a Classic Cloud Service to an Azure SQL Database instance with a Private Endpoint in an Azure RM VNET.

The architecture is described as per the following diagram:

![architecture](https://github.com/arincoau/architecture.png "architecture")

[github repo](https://www.google.com) with code, deployment scripts, arm templates, etc.

## Prerequisites

- Azure Classic CLI - [installation instuctions](https://docs.microsoft.com/en-us/cli/azure/install-classic-cli?view=azure-cli-latest)
- Azure CLI - [installation instructions](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

## Before you start

These instructions assumes you are familiar with the azure cli and azure in general. You will need to login and select the same subscription using azure and azure classic cli. Please refer to their respective documentation on how to achieve this. The Azure location that resources will be deployed into in this example is Australia East. You can adjust this to suit your needs.

## Create Resource Group

First up we need to deploy a resource group which will house all our resources.

``` sh
az group create \
  --name cloud-service-private-link \
  --location australiaeast
```

## Deploy Classic VNET

Next we use the azure classic cli to create a classic VNET. 

``` sh
azure network vnet create \
  --vnet classic-vnet \
  --address-space 172.20.0.0 \
  --cidr 16 \
  --location australiaeast \
  --subnet-name default \
  --subnet-cidr 24 \
  --subnet-start-ip 172.20.1.0
```

**NOTE**: This will only work if you are a co-administrator of the subscription, if not you will need to perform this manually via the portal. Follow these [instructions](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-create-vnet-classic-pportal) with the following details.

- **Name**: classic-vnet
- **Address space**: 172.20.0.0/16
- **Subnet name**: default
- **Subnet address range**: 172.20.1.0/24
- **Subscription**: *Your subscription name*
- **Resource Group**: cloud-service-private-link
- **Location**: Australia East

## Deploy ARM VNET

Now we deploy the ARM VNET.

``` sh
az network vnet create \
  --resource-group cloud-service-private-link \
  --name rg-vnet \
  --location australiaeast \
  --address-prefixes 172.21.0.0/16 \
  --subnet-name default \
  --subnet-prefixes 172.21.1.0/24
  --disable-private-endpoint-network-policies true
```

## Peer ARM VNET to Classic VNET

To peer the ARM VNET to the Classic VNET you will first need to locate the Resource ID of the Classic VNET. You can do this by locating it in the portal (in the properties tab of the classic VNET.) Or you can replace <subscription_id> with you subscription id in the following.

**Resource ID**: /subscriptions/<subscription_id>/resourceGroups/cloud-service-private-link/providers/Microsoft.ClassicNetwork/virtualNetworks/classic-vnet

To get your subscription id execute:

``` sh
az account show \
  --query id \
  --output tsv
```

It should look something like this:
7d74dc30-64d0-4bb7-94cf-db54d4df18d2

Then to peer your classic vnet to your arm vnet execute the following replacing the value for `--remote-vnet`

``` sh
az network vnet peering create \
  --resource-group cloud-service-private-link \
  --name rg-vnet-classic-vnet-peering \
  --vnet-name rg-vnet \
  --remote-vnet /subscriptions/7d74dc30-64d0-4bb7-94cf-db54d4df18d2/resourceGroups/cloud-service-private-link/providers/Microsoft.ClassicNetwork/virtualNetworks/classic-vnet
```

## Deploy DNS Forwarder

Next up we will deploy the DNS forwarder. The DNS forwarder will be the DNS server for the classic VNET, it will forward on any DNS requests to the Azure DNS server of the rg VNET. The Azure DNS server will then in turn resolve the DNS of the Azure SQL Private Link private DNS zone entry.

To start with we need to deploy an availability set for the dns forwarder. Azure makes sure that the VMs you place within an Availability Set run across multiple physical servers, compute racks, storage units, and network switches. If a hardware or software failure happens, only a subset of your VMs are impacted and your overall solution stays operational.

``` sh
az vm availability-set create \
  --resource-group cloud-service-private-link \
  --name dnsproxy-availability-set \
  --location australiaeast \
  --platform-fault-domain-count 2 \
  --platform-update-domain-count 2
```

Next up we will deploy the network interface cards for the Virtual Machines.

``` sh
az network nic create \
  --resource-group cloud-service-private-link \
  --name dnsproxy-vm-nic-1 \
  --location australiaeast \
  --vnet-name rg-vnet \
  --subnet default \
  --private-ip-address 172.21.1.4

az network nic create \
  --resource-group cloud-service-private-link \
  --name dnsproxy-vm-nic-2 \
  --location australiaeast \
  --vnet-name rg-vnet \
  --subnet default \
  --private-ip-address 172.21.1.5
```

And now we can deploy our virtual machines. 

``` sh
az vm create \
  --resource-group cloud-service-private-link \
  --name dnsproxy-vm-1 \
  --location australiaeast \
  --size Standard_A1 \
  --image UbuntuLTS \
  --nics dnsproxy-vm-nic-1 \
  --availability-set dnsproxy-availability-set

az vm create \
  --resource-group cloud-service-private-link \
  --name dnsproxy-vm-2 \
  --location australiaeast \
  --size Standard_A1 \
  --image UbuntuLTS \
  --nics dnsproxy-vm-nic-2 \
  --availability-set dnsproxy-availability-set
```

Now we can install the DNS proxying software. In this instance we will be using Bind9. More information on Bind9 can be found [here](https://wiki.debian.org/Bind9).

To do this we will use a custom script virtual machine extension. The script is available to download [here](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/301-dns-forwarder/forwarderSetup.sh). We will, however, reference it via it's uri. The script take two arguments. The first argument is the IP address of the server that DNS requests should be forwarded to. In our case this is the Azure DNS IP address 168.63.129.16. The second argument is the CIDR notation of client IP address range that the dns server will accept requests from. In our case this is the IP address range of the classic vnet 172.20.0.0/16.

``` sh
az vm extension set \
  --vm-name dnsproxy-vm-1 \
  --resource-group cloud-service-private-link \
  --publisher Microsoft.Azure.Extensions \
  --name CustomScript \
  --version 2.0 \
  --settings '{"fileUris": ["https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/301-dns-forwarder/forwarderSetup.sh"], "commandToExecute":"sh forwarderSetup.sh 168.63.129.16 172.20.0.0/16"}'

az vm extension set \
  --vm-name dnsproxy-vm-2 \
  --resource-group cloud-service-private-link \
  --publisher Microsoft.Azure.Extensions \
  --name CustomScript \
  --version 2.0 \
  --settings '{"fileUris": ["https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/301-dns-forwarder/forwarderSetup.sh"], "commandToExecute":"sh forwarderSetup.sh 168.63.129.16 172.20.0.0/16"}'
```

## Update Classic VNET DNS Server

Now that we have deployed our DNS proxy virtual machines into our rg-vnet we can update the DNS server entries for the classic-vnet. This can be achieved via the portal.

- Navigate to the `cloud-service-private-link` resource group
- Under settings open the DNS Servers pane
- Add two entries for each of the DNS servers in the rg-vnet
  - 172.21.1.4
  - 172.21.1.5
- Click Save

## Deploy an Azure SQL Database with Private Link

First we deploy a SQL server. You should replace the <admin_user> and <admin_password> values in the following command with your own values. Please take note of these values as we will use them later to connect to the database from the cloud service.

``` sh
az sql server create \
  --resource-group cloud-service-private-link \
  --name example-sql-server \
  --admin-user <admin_user> \
  --admin-password '<admin_password>'
```

And now we create a database. In this instance we will use the AdventureWorks example database so that we can validate everything is working with our example cloud service.

``` sh
az sql db create \
  --resource-group cloud-service-private-link \
  --server example-sql-server \
  --name adventure_works \
  --sample-name AdventureWorksLT \
  --tier Basic \
  --capacity 5 \
  --max-size 104857600 \
  --collation SQL_Latin1_General_CP1_CI_AS
```

Before we deploy the private link we need to update the rg-vnet subnet to disable private endpoint network policies. 

``` sh
az network vnet subnet update \
  --resource-group cloud-service-private-link \
  --vnet-name rg-vnet \
  --name default \
  --disable-private-endpoint-network-policies true
```

We need to create a private endpoint in out rg-vnet for our Azure SQL Server, but before we do this we need to get the resource ID of the sql server we created earlier.

``` sh
az sql server show \
  --resource-group cloud-service-private-link \
  --name example-sql-server \
  --output tsv \
  --query id
```

Take note of the output and replace <sql_server_id> in the following command.

``` sh
az network private-endpoint create \
    --name example-sql-private-link \
    --resource-group cloud-service-private-link \
    --vnet-name rg-vnet  \
    --subnet default \
    --private-connection-resource-id "/subscriptions/6de39ed6-398b-446e-944f-b8b5ba08bb58/resourceGroups/cloud-service-private-link/providers/Microsoft.Sql/servers/example-sql-server" \
    --group-ids sqlServer \
    --connection-name example-sql-private-link-connection
```

Now we create the private DNS zone which will house the records that resolve the name of our SQL server to our private IP.

``` sh
az network private-dns zone create \
  --resource-group cloud-service-private-link \
  --name  "privatelink.database.windows.net"
```

And now we link the private DNS zone to our VNET.

``` sh
az network private-dns link vnet create \
  --resource-group cloud-service-private-link \
  --zone-name  "privatelink.database.windows.net" \
  --name rg-vnet-private-dns-link \
  --virtual-network rg-vnet \
  --registration-enabled false
```

Now we need to locate the resource ID of the network interface card that was created by our private link.

``` sh
az network private-endpoint show \
  --name example-sql-private-link \
  --resource-group cloud-service-private-link \
  --query 'networkInterfaces[0].id' \
  --output tsv
```

Take note of the output and use it as the <nic_id> value in the following command to get the IP address of the private link endpoint. Take note of this value as it will be used to create the a records in the private dns zone.

``` sh
az resource show --ids <nic_id> \
  --api-version 2019-04-01 \
  --query 'properties.ipConfigurations[0].properties.privateIPAddress' \
  --output tsv
```

Now we can create the dns zone entries. Replace <private_ip> with the IP address of the private link we identified above.

``` sh
az network private-dns record-set a create \
  --name example-sql-server \
  --zone-name privatelink.database.windows.net \
  --resource-group cloud-service-private-link

az network private-dns record-set a add-record \
  --record-set-name myserver \
  --zone-name privatelink.database.windows.net \
  --resource-group cloud-service-private-link \
  --ipv4-address <private_ip>
```

## Deploy Cloud Service
