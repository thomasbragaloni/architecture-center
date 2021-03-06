---
title: Plan your deployment for updating Windows Virtual Machines in Azure
titleSuffix: Azure Example Scenarios
description: A discussion of how best to configure your environment for Windows Service Update Services
author: vpaulreed
ms.date: 03/15/2020
ms.topic: example-scenario
ms.service: architecture-center
ms.subservice: example-scenario
ms.custom:
  - fcp
---
# Plan your deployment for updating Windows Virtual Machines in Azure

If you locked down your Azure Virtual Network (Microsoft Azure Virtual Network) from the Internet, you can still get Windows Updates without jeopardizing security and opening up access to the Internet as a whole. This article contains recommendations how you can set up a perimeter network, also called a DMZ, to host a Windows Server Update Service (WSUS) instance to securely update VNets without Internet connectivity.

Customers using Azure Firewall can use the Windows Update FQDN tag in application rules to allow the required outbound network traffic through your firewall.  For more information, see [FQDN tags overview](https://docs.microsoft.com/azure/firewall/fqdn-tags).

This article assumes a familiarity with Azure services. The following sections describe the recommended deployment design with a hub and spoke configuration in a single or multi-region configuration.

## Azure Virtual Network hub-spoke network topology

The recommendation is to set up a hub and spoke model network topology by creating a perimeter network, also called a DMZ, host the WSUS Server on an Azure Virtual Machine that is in the hub between the Internet and VNets. The hub will have open ports, WSUS uses port 80 for HTTP protocol and port 443 for HTTPS protocol to obtain updates from Microsoft. The spokes are all of the other VNets, which will communicate with the Hub, and not with the Internet. This will be accomplished by creating a Subnet, Network Security Groups (NSGs), and Azure Microsoft Azure Virtual Network Peering that allows the specific traffic while blocking other Internet traffic. The following image illustrates a sample hub-spoke topology.

![Architecture diagram](.\wsus-vnet.png)

In the image above:

- **DMZSubnet** – The hub in the hub and spoke.
- **NSG_DS** – Network Security Group rule that allows traffic for WSUS while blocking other Internet traffic.
- **WSUS VM** – The Azure Virtual Machine configured to run WSUS.
- **MainSubnet** – A Virtual Network, a spoke, with Virtual Machines.
- **NSG_MS** – Network Security Group policy that allows traffic from WSUS VM, but denies Internet traffic.

You can reuse an existing server, or deploy a new server that will be the WSUS. Minimum recommendation for the WSUS VM:

- **Operating System:** Windows Server 2016 or later
- **Processor:** Dual-Core, 2 GHz or faster
- **Memory:** 2 GB of RAM more than what is required by the server and all other services and software running.
- **Storage:** 40 GB or greater
- To access your Virtual Machine more securely using Just-In-time (JIT), read [Manage virtual machine access using just-in-time](https://docs.microsoft.com/azure/security-center/security-center-just-in-time).

Your network will have more than one Microsoft Azure Virtual Network, which can be in the same or different regions. You'll need to evaluate all Windows Server VMs to see if one can be used as a WSUS. If you have thousands of VMs to update, we recommend having a Windows Server dedicated to the WSUS role.

If all your VNets are in the same region, then we suggest having one WSUS for every 18,000 VMs. This is based on a combination of the VM's requirements based on the number of client VMs being updated, and cost of communicating between VNets. For more information on WSUS capacity requirements, see the article [Determine WSUS Capacity Requirements](https://docs.microsoft.com/security-updates/windowsupdateservices/18127528).

You can find the cost of these configurations by using the [Azure Pricing calculator](https://azure.microsoft.com/pricing/calculator/). You'll need to add the following:

- Virtual Machine
  - Region: Select the region where your Microsoft Azure Virtual Network is currently deployed
  - Operating System: Windows
  - Tier: Standard
  - Instance: D4 configuration
  - Managed Disks: Standard HDD 64 GB
- Virtual Network
  - Type
    - Same Region if transfer is in the same region
    - Across Region if moving data from one region to another
  - Data Transfer 2 GB
  - Region
    - If within the same region, choose the region the WSUS and VNets are in
    - If across region, the Source Microsoft Azure Virtual Network region is where the WSUS is. The destination Microsoft Azure Virtual Network Region is where the data is going
  - If you have multiple regions, you'll need to select virtual network multiple times

It's important to note that prices will vary by region.

## Manual deployment

After you've either identified the Microsoft Azure Virtual Network to use as the Hub or determine that you'll need to create a new Windows Server instance, you'll need to create an NSG rule. The rule will allow Internet Traffic, which allows Windows Update metadata and content to sync with the WSUS you'll create. Here are the rules that need to be added:

- Add inbound/outbound NSG rule to allow traffic to/from the _Internet_ on Port 80 (for content)
- Add inbound/outbound NSG rule to allow traffic to/from the _Internet_ on Port 443 (for metadata)
- Add inbound/outbound NSG rule to allow traffic from the _Client VMs_ on Port 8530 (default unless configured)

## Setting up WSUS

There are two approaches you can use to set up your WSUS server.

- If you want to automatically set up a server configured to handle a typical workload with minimal administration required, you can use the PowerShell automation script.
- If you need to handle thousands of clients running many different operating systems and languages, or if you want to configure WSUS in a way that the PowerShell script can't handle, you can set up WSUS manually. Both approaches are described below.

You can also combine the two approaches, by using the automation script to do most of the work, then using the WSUS administrative console to fine-tune the server settings.

### Setting up WSUS with the automation script

The `Configure-WSUSServer` script allows you to quickly set up a WSUS server that will automatically synchronize and approve updates for a chosen set of products and languages.

>[!NOTE]
>The script always sets up WSUS to use Windows Internal Database to store its update data. This speeds up setup and reduces administration complexity. But if your server will support thousands of client computers, especially if you also need to support a wide variety of products and languages, you should set up WSUS manually instead so that you can use SQL Server as the database.

The latest version of this script is [available on GitHub](https://github.com/mspnp/solution-architectures/tree/master/wsus).

The script is configured through a JSON file. The following options can currently be configured:

- Whether update payloads should be stored locally (and if so, where they should be stored), or left on the Microsoft servers
- Which products, update classifications, and languages should be available on the server
- Whether the server should automatically approve updates for installation, or leave updates unapproved unless an administrator approves them
- Whether the server should automatically retrieve new updates from Microsoft, and if so, how often
- Whether express update packages should be used. (Express update packages reduce server-to-client bandwidth, at the expense of client CPU/disk usage and server-to-server bandwidth.)
- Whether the script should overwrite its previous settings. (Normally, to avoid inadvertent reconfiguration that might disrupt server operation, the script will only run once on a given server.)

Copy the script and its configuration file to local storage, and edit the configuration file to suit your needs.

>[!WARNING]
>Be careful when editing the configuration file. The syntax used for JSON configuration files is very strict; if you inadvertently modify the structure of the file rather than just the parameter values, the configuration file will fail to load.

This script can be run in one of two ways:

- You can run the script manually, from the WSUS VM
  - This command, run from an elevated Command Prompt window, will install and configure WSUS, using the script and configuration file in the current directory:

  `powershell.exe -ExecutionPolicy Unrestricted -File .\Configure-WSUSServer.ps1 -WSUSConfigJson .\WSUS-Config.json`

- You can use the [Custom Script Azure VM Extension](https://docs.microsoft.com/azure/virtual-machines/extensions/custom-script-windows)
  - Copy the script and the JSON configuration file to your own storage container
  - In typical VM and Microsoft Azure Virtual Network configurations, the Custom Script extension needs only these two parameters to run the script correctly (substituting in the correct URL for your storage locations):

  ```json
    "fileUris": ["https://mystorage.blob.core.windows.net/mycontainer/Configure-WSUSServer.ps1","https://mystorage.blob.core.windows.net/container/WSUS-Config.json"],
    "commandToExecute": "powershell.exe -ExecutionPolicy Unrestricted -File .\Configure-WSUSServer.ps1 -WSUSConfigJson .\WSUS-Config.json"

  ```

The script will start the initial synchronization needed to make updates available to client computers. However, it won't wait for that synchronization to complete. Depending on the products, classifications, and languages you've selected, the initial synchronization may take several hours to complete. All synchronizations after that should be shorter.

### Setting up WSUS manually

1. **From your WSUS VM**, open _Server Manager_ and click _Add roles and features_.
2. Click _Next_ until you get to the _Select server roles_ page and select _Windows Server Update Services_. Click _Add Features_ when you're prompted "Add features that are required for Windows Server Update Services?"
3. Click _Next_ until you get to the _Select role services_ page.
    - By default, you can use _WID Connectivity_.
    - Use _SQL Server Connectivity_ if you need to support clients with many different Windows versions (i.e. Windows 10, Windows 8, Windows 7, etc.).

4. Click _Next_ until you get to the _Content location selection_ page. Type in where you want to store the updates.
5. Click _Next_ until you get to the _Confirm installation selections_ page. Click _Install_.
6. Open the installed _Windows server Update Services_ and click _Run_.
7. Click _Next_ until you get to the _Connect to Upstream Server_ page and click _Start Connecting_.
8. Click _Next_ until you get to the _Choose Languages_ page. Choose languages you need.
9. Click _Next_ until you get to the _Choose Products_ page. Choose products you need.
10. Click _Next_ until you get to the _Choose Classifications_ page. Choose updates you need.
11. Click _Next_ until you get to the _Set Sync Schedule_ page. Choose your sync preference.
12. Click _Next_ until you get to the _Finished_ page. Click _Begin initial synchronization_ and click _Next_.
13. Click _Next_ until you get to the _What's Next_ page and click _Finish_.
14. If you click on your _WSUS name_ (i.e. WsusVM) in the navigation pane, you should see that _Synchronization Status_ is _Idle_ and _Last synchronization result_ is _Succeeded_.
15. In the navigation pane, click on _Options_ → _Computers_ → _Use Group Policy or registry settings on computers_. Click _OK_.

During synchronization, WSUS determines if any new updates have been made available since the last time you synchronized. If it is your first time synchronizing WSUS, the metadata is downloaded immediately. The payload isn't downloaded unless local storage is turned on and the update is approved for at least one computer group.

>[!NOTE]
>Initial synchronization can take over an hour. All synchronizations after that should be significantly shorter.

## Configure virtual networks to communicate with WSUS

Next set up Azure Microsoft Azure Virtual Network Peering or Global Microsoft Azure Virtual Network Peering to communicate with the Hub. We recommend that you set up a WSUS in each region that you've deployed to minimize latency.

On each Microsoft Azure Virtual Network that is the spoke, you'll need to create an NSG Policy with the following rules:

- Add inbound/outbound NSG rule to allow traffic from the _WSUS VM_ on Port 8530 (default unless configured).
- Add inbound/outbound NSG rule to deny traffic from the _Internet_.

Next create the Microsoft Azure Virtual Network Peering from the spoke to the Hub.

### Client VM

- For extra security, you can get rid of the VM's associated public IP address. Read [View, change settings for, or delete a public IP address](https://docs.microsoft.com/azure/virtual-network/virtual-network-public-ip-address#view-change-settings-for-or-delete-a-public-ip-address).
- To access your Virtual Machine more securely using JIT, read [Manage virtual machine access using just-in-time](https://docs.microsoft.com/azure/security-center/security-center-just-in-time).

## Configure client virtual machines

Any Virtual Machines running Microsoft Windows (except Home SKU) can be updated using WSUS. Go to each client Virtual machine and do the following steps to enable communication between the WSUS and client:

### From your client VM

1. Open _Local Group Policy Editor_ (or Group Policy Management Editor)
2. Click _Computer Configuration_ → _Administrative Templates_ → _Windows Components_ → _Windows Update_
3. Enable _Specify intranet Microsoft update service location_
4. Enter in the URL – `http://[WSUS name]:8530`. (You can see your _WSUS name_ (i.e. WsusVM) from the _Update Services_ page.) It may take some time (up to few hours) for this setting to be reflected.
5. Go to _Settings_ → _Update & Security_ → _Windows Update_
6. Click _check for updates_

### From your WSUS VM

1. Open _Windows server Update Services_. You should be able to see your client VM listed under _Computers_ → _All Computers_
2. Click on _Updates_ → _All Updates_
3. Set _Approval_ as _Any Except Declined_
4. Set _Status_ as _Needed_. Now, you can see all the needed updates for your client VM.
5. Right click on any of the updates and _approve_ installation.

### Verification

1. On the client VM, go to _Settings_ → _Update & Security_ → _Windows Update_
2. Click _check for updates_. You should see an update with the same KB article number (i.e. 4480056) which you approved from the WSUS VM.

Administrators managing a large network should see the article [Configure automatic updates and update service location](https://docs.microsoft.com/windows/deployment/update/waas-manage-updates-wsus#configure-automatic-updates-and-update-service-location) to use Group Policy settings to automatically configure clients.

## WSUS deployment for multiple clouds

It isn't possible to set up Virtual Network peering across public and private clouds. Networks that are deployed across public and private clouds will need to have at least one WSUS set up in each cloud.

## Support notes

Currently, WSUS doesn't support syncing with Windows Home Sku.

## Azure Update Management

You can use the Update Management solution in Azure to manage and schedule operating system updates for VMs that are syncing against WSUS. The patch status of the VM (i.e. which patches are missing) is assessed based on the source that the VM is configured to sync with. If the Windows VM is configured to report to WSUS, depending on when WSUS last synced with Microsoft Update, the results might differ from what Microsoft Update shows. Once you have finished configuring your WSUS environment, you can enable Update Management. For more information, see the [Update Management overview and Onboarding steps](https://docs.microsoft.com/azure/automation/automation-update-management).

## Next steps

- For more information on managing WSUS, setting up a WSUS synchronization schedule and more, please see [WSUS Administration](https://docs.microsoft.com/windows-server/administration/windows-server-update-services/get-started/windows-server-update-services-wsus).
