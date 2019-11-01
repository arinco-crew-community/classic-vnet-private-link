# Connect a Cloud Service to a Azure SQL DB using Private Link (Preview)

This blog post details how to use az and azure cli to connect a Classic Cloud Service to an Azure SQL Database instance with a Private Endpoint in an Azure RM VNET.

The architecture is described as per the following diagram:

![architecture](https://github.com/arincoau/architecture.png "architecture")

[github repo](https://www.google.com) with code, deployment scripts, arm templates, etc.

## Prerequisites

- Azure Classic CLI - [installation instuctions](https://docs.microsoft.com/en-us/cli/azure/install-classic-cli?view=azure-cli-latest)
- Azure CLI - [installation instructions](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

## Before you start

These instructions assumes you are familiar with the azure cli and azure in general. You will need to login and select the same subscription using azure and azure classic cli. Please refer to their respective documentation on how to achieve this. The Azure location resources will be deployed into in this example is Australia East. You can adjust this to suit your needs.

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
```

## Peer ARM VNET to Classic VNET

To peer the ARM VNET to the Classic VNET you will first need to locate the Resource ID of the Classic VNET. You can do this by locating it in the portal (in the properties tab of the classic VNET.) Or you can replace <subscription_id> with you subscription id in the following.

**Resource ID**: /subscriptions/<subscription_id>/resourceGroups/cloud-service-private-link/providers/Microsoft.ClassicNetwork/virtualNetworks/classic-vnet

To get your subscription id execute:


It should look something like this: 

## Deploy DNS Forwarder

## Update Classic VNET DNS Server

## Deploy Azure SQL with Private Link

## Deploy Cloud Service
