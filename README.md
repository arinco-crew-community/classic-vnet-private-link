# Connect a Azure Classic Virtual Machine to a Azure SQL DB using Private Link (Preview)

This blog post details how to deploy and connect a classic virtual machine in a classic virtual network to an Azure SQL Database instance with a Private Endpoint in an Azure RM VNET.

The architecture is described as per the following diagram:

![architecture](https://raw.githubusercontent.com/arincoau/classic-vnet-private-link/master/arch_diagram.png "architecture")

If you'd like to skip the instructional part of the blog post you can find the [github repo](https://www.github.com/arincoau/classic-vnet-private-link/) with arm templates to achieve the same result [here](https://www.github.com/arincoau/classic-vnet-private-link/). **NOTE**: You will still have to manually deploy the Azure Classic Virtual Machine.

## Prerequisites

- Azure CLI - [installation instructions](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

## Before you start

These instructions assumes you are familiar with the Azure CLI and have some general Azure knowledge. The Azure location that resources will be deployed into in this example is Australia East, you can adjust this location to suit your needs, but it will need to be the same for all resources.

## Create Resource Group

Before we start deploying any resource we need to deploy a resource group which will house all our resources.

``` sh
az group create \
  --name cloud-service-private-link \
  --location australiaeast
```

## Deploy Classic VNET

The first thing you will need to do is deploy a classic VNET via the Azure portal. This can be done by completing the following steps:

1. In the Azure portal Navigate to the `cloud-service-private-link` resource group
1. Click **Add**
1. Search for **Virtual Network**
1. Click the **(change to Classic)** link next to **Deploy with Resource Manager**
1. Click **Create**
1. Fill the form with the following values:

- **Name**: classic-vnet
- **Address space**: 172.20.0.0/16
- **Subnet name**: default
- **Subnet address range**: 172.20.0.0/24
- **Subscription**: *Your subscription name*
- **Resource Group**: cloud-service-private-link
- **Location**: Australia East

7. Click **Create**

## Deploy ARM VNET

Now we deploy the ARM VNET.

``` sh
az network vnet create \
  --resource-group cloud-service-private-link \
  --name rm-vnet \
  --location australiaeast \
  --address-prefixes 172.21.0.0/16 \
  --subnet-name default \
  --subnet-prefixes 172.21.0.0/24
```

## Peer ARM VNET to Classic VNET

To peer the ARM VNET to the Classic VNET you will first need to locate the Resource ID of the Classic VNET. You can do this by locating it in the portal (in the properties tab of the classic VNET.)

The resource ID will look like this:
/subscriptions/5ef11cc0-15c0-4f78-a5f8-c8053e813f30/resourceGroups/cloud-service-private-link/providers/Microsoft.ClassicNetwork/virtualNetworks/classic-vnet

Then to peer your classic vnet to your arm vnet execute the following replacing the value for <remote_vnet_resource_id> with the remote vnet resource id idenitfied above.

``` sh
az network vnet peering create \
  --resource-group cloud-service-private-link \
  --name rm-vnet-classic-vnet-peering \
  --vnet-name rm-vnet \
  --allow-vnet-access \
  --remote-vnet <remote_vnet_resource_id>
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
  --vnet-name rm-vnet \
  --subnet default \
  --private-ip-address 172.21.0.4

az network nic create \
  --resource-group cloud-service-private-link \
  --name dnsproxy-vm-nic-2 \
  --location australiaeast \
  --vnet-name rm-vnet \
  --subnet default \
  --private-ip-address 172.21.0.5
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

To do this we will use a custom script virtual machine extension. The script is available to download [here](https://raw.githubusercontent.com/arincoau/classic-vnet-private-link/master/forwarderSetup.sh). We will, however, reference it via it's uri. The script take two arguments. The first argument is the IP address of the server that DNS requests should be forwarded to. In our case this is the Azure DNS IP address 168.63.129.16. The second argument is the CIDR notation of client IP address range that the dns server will accept requests from. In our case this is the IP address range of the classic vnet 172.20.0.0/16.

``` sh
az vm extension set \
  --vm-name dnsproxy-vm-1 \
  --resource-group cloud-service-private-link \
  --publisher Microsoft.Azure.Extensions \
  --name CustomScript \
  --version 2.0 \
  --settings '{"fileUris": ["https://raw.githubusercontent.com/arincoau/classic-vnet-private-link/master/forwarderSetup.sh"], "commandToExecute":"sh forwarderSetup.sh 168.63.129.16 172.20.0.0/16"}'

az vm extension set \
  --vm-name dnsproxy-vm-2 \
  --resource-group cloud-service-private-link \
  --publisher Microsoft.Azure.Extensions \
  --name CustomScript \
  --version 2.0 \
  --settings '{"fileUris": ["https://raw.githubusercontent.com/arincoau/classic-vnet-private-link/master/forwarderSetup.sh"], "commandToExecute":"sh forwarderSetup.sh 168.63.129.16 172.20.0.0/16"}'
```

## Update Classic VNET DNS Server

Now that we have deployed our DNS proxy virtual machines into our rm-vnet we can update the DNS server entries for the classic-vnet. This can be achieved via the portal.

1. In the Azure portal Navigate to the `cloud-service-private-link` resource group
2. Open the `classic-vnet`
3. Under settings open the DNS Servers pane
4. Add two entries for each of the DNS servers in the rm-vnet

- 172.21.0.4
- 172.21.0.5

5. Click Save

## Deploy an Azure SQL Database

Now we can deploy the SQL server. You should replace the <admin_user> and <admin_password> values in the following command with your own values. Please take note of these values as we will use them later to connect to the database from the cloud service.

``` sh
az sql server create \
  --resource-group cloud-service-private-link \
  --name example-sql-server \
  --admin-user clancy \
  --admin-password '<admin_password>'
```

## Deploy Azure Private Link

Before we deploy the private link we need to update the rm-vnet default subnet to disable private endpoint network policies.

``` sh
az network vnet subnet update \
  --resource-group cloud-service-private-link \
  --vnet-name rm-vnet \
  --name default \
  --disable-private-endpoint-network-policies true
```

We need to create a private endpoint in out rm-vnet for our Azure SQL Server, but before we do this we need to get the resource ID of the sql server we created earlier.

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
    --vnet-name rm-vnet  \
    --subnet default \
    --private-connection-resource-id /subscriptions/9b5fe952-db67-415c-8d69-f9beaa466249/resourceGroups/cloud-service-private-link/providers/Microsoft.Sql/servers/example-sql-server \
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
  --name rm-vnet-private-dns-link \
  --virtual-network rm-vnet \
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
  --record-set-name example-sql-server \
  --zone-name privatelink.database.windows.net \
  --resource-group cloud-service-private-link \
  --ipv4-address <private_ip>
```

## Deploy Classic Virtual Machine

We can now deploy the classic virtual machine and test out connectivity to our Azure SQL Database. We will need do this via the Azure portal.

1. In the Azure portal Navigate to the `cloud-service-private-link` resource group
1. Click **Add**
1. Search for **Virtual Network**
1. Click the **(change to Classic)** link next to **Deploy with Resource Manager**
1. Leave the deault software plan selected
1. Click **Create**
1. Enter the following Details:

- **Name**: sqltestserver
- **Username**: *Enter a username*
- **Password**: *Enter a password*
- **Confirm Password**: *Enter a password*
- **Subscription**: *Name of your subscription*
- **Resource Grouo**: cloud-service-private-link
- **Location**: Australia East

8. Click Next
9. Select a small VM size (I chose DS1_V2)
10. Change storage type to Standard
11. Under network select in **Cloud service (domain name)**
12. Click **Create new**
13. Enter a **Domain name** for the virtual machine
14. Click **OK**
15. Ensure the other network settings match the following:

- **Virtual Network**: classic-vnet
- **Subnet**: default (172.20.0.0/24)
- **Private IP address**: Dynamic
- **Dynamic IP address**: Dynamic

16. Click **Endpoints**
17. Click **+ Add an endpoint**
18. Enter the following details:

- **Name**: RDP
- **Protocol**: TCP
- **Public Port**: 3389
- **Private Port**: 3389
- **Floating IP address**: Disabled

19. Click OK, OK, OK, OK
20. Wait for the deployment to complete and the virtual machine to be in the *running* state

## Validate connectivity between Classic Virtual Machine and Azure SQL Database

To validate connectivity between the classic virtual machine and the Azure SQL database you will need to RDP into the virtual machine.

1. In the Azure portal Navigate to the `cloud-service-private-link` resource group
1. Open the Cloud service with the **Domain name** you specified in the steps above.
1. Open **Roles and Instances**
1. Select the server with the name given above
1. Click **Connect** (This will download an RDP file)
1. Open the RDP file and connect to the virtual machine using the username and password specified when creating the virtual machine

Once you have sucessfully connected to the virtual machine you can try pinging the private DNS of the Azure SQL Private Link. `example-sql-server.privatelink.database.windows.net` it should resolve to `172.21.0.4`.

Download and install SQL Server Management Studio on the virtual machine using this link [https://aka.ms/ssmsfullsetup](https://aka.ms/ssmsfullsetup).

Open SQL Server Management Studio and enter the database server as `example-sql-server.privatelink.database.windows.net` and the username and password you specified for the SQL sever above. You should be able to successfully connect. 