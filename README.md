# Networking-fundamentals-in-Azure
## How to Set-Up a Network Environment and Create a Firewall in Azure Using PowerShell
### In this lab, we will be go through the following steps:

1. Setting up a network environment in Azure with PowerShell

2. Deploy a firewall

3. Create a default route

4. Configure an application rule to allow access to google.com

5. Configure a Network rule to allow access to external DNS Servers

6. Test the Firewall

7. Cleanup Resources
Setting up a network environment in Azure with PowerShell


##### To communicate with Azure subscription with Powershell Deskop for the first time, use the following steps:
1. Open Powershell Desktop as Administrator
2. Set the Execution Policy (if you haven't already) to allow scripts to run using the code:
###### Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
3. Install the AZ module from the Powershell gallery:
###### Install-Module -Name Az -Scope CurrentUser -Repository PSGallery -Force
4. Connect interactively to your Azure account using:
###### Connect to Your Azure Account


### Network Environment
#### In this network environment, we will create three subnets that will be housing the workloads, Azure Bastion and Azure Firewall. This will make the environment look like the diagram shown above.

I. The first thing to create before creating the virtual network is a resource group. All the resources that will be created for this lab will be under the same resource group.

###### New-AZResourceGroup -Name Lab-Resources -Location eastus

II. The next step is to create the Virtual Network and subnets. In order to do this, we would first define our three subnets and store them into the PowerShell variable as shown in the code below. (Beware of the special subnet name for Bastion and Firewall subnets).

###### $Bastion = New-AzVirtualNetworkSubnetConfig -Name AzureBastionSubnet -AddressPrefix 10.0.0.0/27
###### $FW_Subnet = New-AzVirtualNetworkSubnetConfig -Name AzureFirewallSubnet -AddressPrefix 10.0.0.0/26
###### $Workload_Subnet = New-AzVirtualNetworkSubnetConfig -Name Workload-Subnet -AddressPrefix 10.0.0.0/24

III. Creating a virtual Network using the established subnets
###### $Lab_VN = New-AzVirtualNetwork -Name Lab-VN -ResourceGroupName Lab-Resources -Location "East US" -AddressPrefix 10.0.0.0/16 -Subnet $Bastion, $FW_Subnet, $Workload_Subnet


IV. We need to create a host for the Bastion resource but we will first create a public IP (PIP) for the Azure Bastion host because it is an internet facing resource (The essence of Azure Bastion is to provide a secure, managed, and seamless way to access virtual machines (VMs) in Azure, dramatically reducing your security exposure. Think of Azure Bastion as a managed Jump Box or Jump Server service provided by Microsoft).

###### $Public_Ip = New-AzPublicIpAddress -ResourceGroupName Lab-Resources -Loaction eastus -Name Bastion-pip -AllocationMethod static -SKU standard 


V. The Bastion will now be created using the created PIP
###### New-AzBastion -ResourceGroupName Lab-Resources -Name Bastion-001 -VirtualNetwork $Lab_VN -PublicIpAddress $Public_Ip

VI. Next we will create the Workload Host using splatting method in the PowerShell to define multiple attributes of the VM
###### $vm_params =@{ ResourceGroupName = "Lab-Resources" Location = "East US" VirtualNetworkName = "Lab-VN" SubnetName = "Workload-Subnet" Name = "Workload-VM" Image = "Win2016Datacenter" Size = "Standard_D2s_v3"}

###### $workload_vm = New-AzVM @vm_params


VII. We need to create a network interface Card (NIC) for the workload host. 
a. Get the workload subnet into a variable
###### $Wrkld_sub = Get-AzVirtualNetworkSubnetConfig -Name Workload-Subnet -VirtualNetwork $Lab_VN
b. Creat the NCI
###### $NIC01 = New-AzNetworkInterface -ResourceGroupName Lab-Resources -Name Workload-NI -Subnet $Wrkld_sub -Location eastus 


VIII. After creating the network interface card for the workload host, it is important to add the NIC to the host.
###### Add-AzVMNetworkInterface -VM $workload_vm -Id $NIC01.Id


2. Deploy a Firewall.

In this lab, we are deploying a firewall to control the outbound traffic. An outbound traffic is the traffic leaving our created network to the internet. It is ideal to create a Public IP Address for the firewall as it is where all the outbound traffic from the network would be routed to the internet.

Press enter or click to view image in full size

3. For us to create a default route, we need to create a route table, create a route and associate the routable to a subnet

I. Create a route table


II. Now that a route table has been created, we need to create a route. Before we create a route, we need to get the Private IP Address of the Firewall. This will be used as the NextHopIPAddress in the route.


Getting the Private IP Address of the AzureFirewall

Creating a Route using the created RouteTable
III. Now that we have created a route table and a route, it is time to associate the route table to the appropriate subnet which is the workload subnet.


4. Configure the Firewall Application Rule. It is important to note that the Application rule allows or deny outbound traffic based on OSI model Layer 7. It is used to filter traffic based on FQDN, URL, HTTP and HTTPS.


5. Configure the Firewall Network Rule. Network rule allows or deny inbound and outbound traffic based on the network layer (OSI model-layer3) and transport layer (OSI model-layer4). A network rule can be used to filter traffic based on IP address, any ports and any protocols. In this scenario, we want to allow a UDP protocol from the workload subnet to the Network Interface DNS Address.


· For testing purposes, we need to configure the DNS Server of the Network Interface Card (NIC) to the Destination address used in the Network Rule

Press enter or click to view image in full size

6. Test the Firewall.

I. First we will access the workload VM via the Bastion in the Azure GUI


II. We run a lookup of the two uri (google.com and facebook.com) to see if it will pass through the DNS Server in the Bastion VM PowerShell window


III. Then we invoked the web-request of both google.com and facebook.com. It was established that google.com succeeded but facebook.com failed as it is only google.com that was allowed as the target FQDN in the Application Rule.


google.com invoked web request

facebook.com invoked web request
7. Destroy Resources


It is a good practice to destroy your resources when they are not in use. The best way to destroy all resources completely is to delete the Resource Group as all the created resources in this Lab was created in the same Resource Group. The Wait-Job command will ensure that the PowerShell Window does not stop while deleting the Resources.
