# Azure Virtual Network Architectures with 3rd party NVA
## Scenario
Azure customers have varied security requirements. A common pattern for some customers is to have traffic between Virtual Networks (VNETs) and OnPrem be inspected by a virtual firewall in Azure, while have VNET to VNET traffic bypass any such inspection. There are several design options for this security pattern when the firewall is a 3rd Party Network Virtual Appliance (NVA).
## Option 1: Hub-Spoke with NVA in Hub, with full mesh among Spoke VNETs
The Hub-Spoke Design with NVA in Hub is well understood and clearly documented in the [Cloud Adoption Framework](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/hub-spoke-network-topology) and in the [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/dmz/nva-ha). The slight difference in this option is all VNETs are also directly peered amongst each other to allow direct communication without passing through firewall. The topology looks as follows:

![Option 1](/Images/Option1.png)

For VNET to OnPrem traffic, User Defined Routes (UDRs) are required at the Spoke VNETs to ensure traffic is forwarded to the NVA. In order to ensure symmetry for return traffic across stateful firewalls, UDRs are also required at the GatewaySubnet pointing to the NVA (or the ILB front-ending the NVA in an HA configuration) for the Spoke VNET prefix destinations.

The advantage of this option is it is easy to understand. The disadvantage is the roughly n^2 (n-factorial) number of peers which are required to build the full mesh, and possibly hitting the peering [limits](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-resource-manager-virtual-networking-limits).
## Option 2: Virtual WAN for direct communication between VNETs, and Classic Hub to OnPrem
With this option, the Spoke VNETs are connected to both a Virtual WAN Hub as well as a Classic Hub hosting a Virtual Network Gateway (VNG). The Virtual WAN connectivity replaces the full mesh from the previous option. Communication to on-prem is facilitated via the Classic Hub. The topology looks as follows:

![Option 2](/Images/Option2.png)

One important point to understand is an Azure VNET can use only one gateway to learn remote routes. The gateway can be either local to the VNET or via a remote peer. Since VWAN has a “gateway” service, specifically the Route Service gateway, VWAN becomes the Spoke VNETs’ (remote) gateway. This implies the Spoke to the Classic Hub cannot use the “Use Remote Gateway” option on the peer link because the Spoke already has a gateway (in VWAN). 

The VNETs will learn about each other's prefixes and will communicate with each other via the vHub. All the VNET Connections have Association and Propagation to the Default route table (or default label) in order to simplify the configuration. This ensures any-to-any among Spoke VNETs without traversing any NVA firewall.

To reach OnPrem, UDRs are defined at the Spoke VNET subnets for the OnPrem prefixes pointing to the NVA (or an ILB front-ending the NVA in an HA configuration). The NVA will analyze the traffic against the firewalling rules and then forward the traffic out the appropriate vNIC based on its policy. The vNIC resides in the Classic Hub which has learned the OnPrem routes via the VNG and will thus forward packets appropriately to OnPrem. 

For return traffic from OnPrem to Spoke, an [Azure Route Server](https://docs.microsoft.com/en-us/azure/route-server/overview) (ARS) in the Classic Hub is required for OnPrem to learn the Spoke prefixes. If the NVA speaks BGP, it can originate a supernet of all the Spokes. This supernet aggregate will be reflected by the ARS to the VNG to OnPrem. This supernet is purely defined and propagated for purposes of ensuring return traffic does not blackhole, especially when the OnPrem connectivity is ExpressRoute. (If it is VPN, it is possible to configure static routing on the CE.) As in previous Option, in order to ensure symmetry for return traffic, UDRs are also required at the GatewaySubnet pointing to the NVA (or the ILB front-ending the NVA in an HA configuration).

The advantage of this option is there is separation of Hub functions: a vHub for a simple VNET-to-VNET any-to-any communication, and a Classic Hub for traffic to OnPrem. Furthermore, this option eliminates the roughly n^2 peering from the previous option. The disadvantage is the introduction of a new function ARS, and the NVA would need to enable BGP to peer with ARS and originate the Spoke supernet.

## Option 3: Virtual WAN Custom Route Tables
In this option, Virtual WAN Custom Route Tables are used as described [here](https://docs.microsoft.com/en-us/azure/virtual-wan/scenario-route-through-nvas-custom#alternate). The vHub has peering Connections built not only to the Spoke VNETs, but it also has peering Connection built to a DMZ VNET. The 3rd party NVA resides in the DMZ VNET. This topology is illustrated as follows:

![Option 3](/Images/Option3.png)

In vHub, a “Default” and a “None” route table are predefined. For this option, two additional Route Tables are created: the DMZ Route Table (RT-DMZ) and VNET Route Table (RT-VNET). By configuring appropriate Propagation and Association on the Connections, the VNETs do not learn the OnPrem prefixes directly, but only via defined Static Routes pointing to the DMZ Connection with the NVA (or ILB front-ending NVAs) as Next Hop. Similarly, OnPrem reaches the Spoke VNETs via a defined Static Route pointing to the DMZ Connection with the NVA (or ILB front-ending NVAs) as Next Hop. 

The DMZ, on the other hand, is learning both OnPrem and Spoke VNET prefixes and acts as the bridge between VNET and OnPrem, hence satisfying the requirement of filtering traffic between VNET to OnPrem.

The Association on a Connection may be viewed as what the VNET (Connection) will learn, and Propagation on a Connection may be thought of as what the VNET (Connection) will advertise its existence to. For example, if Spoke-VNET1 Connection is propagating to RT-DMZ Route Table, that means anything associated with RT-DMZ will learn Spoke-VNET1’s prefix. Similarly, if Spoke-VNET1 is associated with RT-VNET, it will learn of the prefixes that are propagating to RT-VNET.

The Assocation and Propagation for this Option are defined as follows:

![Association and Propatation Definitions](/Images/Option3-AssocProp.png)

Note, the VNET-Spoke connections are propagating to RT-DMZ so the DMZ can learn its prefixes. It is also propagating to other Spoke VNETs associated with the same Route Table RT-VNET. This allows a Spoke to be able to communicate with another Spoke without traversing the NVA and free of filtering/inspection. 

Similarly, the Branches are Propagating to Default, meaning all VPN branches Associated with the Default route table, connected to the local vHub or a remote vHub, will be able to communicate with each other without traversing the NVA. The Branches are also Propagating to RT-DMZ, so the DMZ learns of Branches.

Because DMZ Connection is propagating to both RT-VNET and Default, both VNETs and Branch can learn about the DMZ prefix and can thus reach the NVA. Once Static Routes are created, pointing to the Prefixes with NextHop of NVA IP (or ILB front-ending the NVA), we will have satisfied the requirement of forcing Spoke to OnPrem traffic through a 3rd-Party NVA. The Static Routes look as follows:

![Static Route Defintions](/Images/Option3-StaticRoutes.png)

The advantage of Option 3 are fewer components (e.g. no ARS as in Option 2) and fewer peerings (e.g. Option 1). The disadvantage is multiple route tables need to be maintained on the Virtual Hub, and it is very important to understand the concept of Association and Propagation for this Option to function.  

## Option 4: Virtual WAN with integrated 3rd-Party NVAs (Preview as of February 2022)

Many partners have plans to integrate their NVAs within the vHub. As of February 2022, Fortigate Firewalls are in preview. Others are targeted in calendar 2022.

## Appendix A: Testing the Options
Options 1 through 3 were validated in simplified lab scenario (using VPN gateway to a single branch). It is of course always recommended you test in your environment, and below are some tips with relevant screen captures to help you get started:

### Option 1: Hub-Spoke with NVA in Hub, with full mesh among Spoke VNETs
This is a well documented scenario. Refer to the documentation in [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/dmz/nva-ha).

### Option 2: Virtual WAN for direct communication between VNETs, and Classic Hub to OnPrem

Below is a simple scenario with a local Spoke VNET peered to both a VHUB and a Classic Hub. The 3rd party NVA is a Cisco CSR. OnPrem is simulated also with CSR terminating via IPSec VPN. 

![Option 2](/Images/Option2-Lab.png)

The VHUB connections are shown as follows, with Association and Propagation to the Default Route Table and Label, or Any to Any. This enables all VNETs (local and remote) to learn of each other. 

![Option 2 Connections](/Images/2-vHubConnections.png)

Effective Route Table on VM in East-Spoke shows peering to the local VHUB and the Classic Hub, and shows Remote VNET being learned by the VHUB. It shows UDRs pointing to the NVA in the Classic Hub (172.22.2.4) to destination OnPrem prefixes 10.0.0.0/8 and 192.168.100.0/24. 

![Option 2 VMRoutes](/Images/2-VMEffectiveRoutes.png)

The below ping and traceroute shows reachability between VMs in Spokes (VM-East 172.17.1.4 to VM-West 172.24.1.4), via vHub:

![Option 2 PingTrace](/Images/2-PingTrace.png)

The NVA, or CSR, peers to Azure Route Server and learns OnPrem via Route Server (AS 65001). It is also sourcing/originating a Supernet 172.16.0.0/13, representing the aggregate of East Region Hub and Spoke.

![Option 2 NVARoutes](/Images/2-NVARoutes.png)

The OnPrem (also a CSR) has learned the Aggregate prefix representing the Local Spokes, or 172.16.0.0/13

![Option 2 OnPremRoutes](/Images/2-OnPremRoutes.png)

SSH validates traffic from SpokeVM to OnPrem.

![Option 2 Verify](/Images/2-Verify.png)

### Option 3: Virtual WAN Custom Route Tables

As explained above, the key to this Option is to ensure the appropriate configuration of Propagation and Association for the Spoke VNETs, OnPrem Branch connection, and DMZ. This ensures Spoke VNETs and Branch do not dynamically learn about each other. Static Routes are defined in the respective route tables (RT-VNET and Default) to ensure traffic hits the DMZ. Below is a simple scenario with a Spoke VNET and DMZ VNET both peered to vHUB, where their Connections are Associated  Custom Route Tables. The 3rd party NVA is a Cisco CSR. OnPrem is simulated also with CSR terminating via IPSec VPN. 

![Option 3](/Images/Option3-Lab.png)

The Spoke and DMZ VNET Connections' Association and Propagation are shown below:

![Option 3 Connections](/Images/3-Connections.png)

Because of the tight control of Propagation, prior to Static Routes being defined, Branch and VNETs cannot reach other, though they can both reach the NVA. Below demonstrates how a Spoke VM can reach a Remote Spoke VM (172.24.1.4) and the DMZ NVA (172.22.2.4) but cannot reach OnPrem (10.3.1.4):

![Option 3 Prior-VMtoOnPrem](/Images/3-Prior-VMtoOnPrem.png)

Below demonstrates how OnPrem can reach the DMZ NVA (172.22.2.4) but cannot reach the Spoke VM (172.17.1.4)

![Option 3 Prior-OnPremtoVM](/Images/3-Prior-OnPremtoVM.png)

Snapshots of the vHub Route Tables confirm  the RT-DMZ Custom Route Table can see both OnPrem (10 and 192 networks) and Spoke (172.17.0.0/22). However, the VNET Custom Route Table RT-VNET and Default Route Table can see only itself and the DMZ prefix BUT not each other.  

![Option 3 Prior-RT-DMZ](/Images/3-Prior-RT-DMZ.png)

![Option 3 Prior-RT-VNET](/Images/3-Prior-RT-VNET.png)

![Option 3 Prior-RT-Default](/Images/3-Prior-RT-Default.png)

Static Routes are now defined in the RT-VNET Custom Route Table for OnPrem prefixes, pointing to the DMZ Connection. Similarly, Static Route is defined in the Default Route Table for the VNET aggregate prefix, pointing to the DMZ Connection. Once these Static Routes are defined, Spoke VNET will have reachability to OnPrem via DMZ. Similarly, OnPrem will have reachability to the Spoke VNET via DMZ. Below shows the Next Hop IP for Static Route:

![Option 3 Static-Route](/Images/3-StaticRoute.png)

With the Static Routes defined in the RT-VNET Custom Route Table for the OnPrem prefixes, RT-VNET custom route table now shows the entries, and the Spoke (associated with RT-VNET) will now have reachability to OnPrem via DMZ.

![Option 3 After-RT-VNET](/Images/3-After-RT-VNET.png)

Similarly, once the Static Route is defined in the Default Route Table for the VNET supernet, the Default Route Table now shows that entry in the Route Table, and OnPrem now has reachability to the VNETs via DMZ.

![Option 3 After-RT-Default](/Images/3-After-RT-Default.png)

Traceroute validates OnPrem to VNET is going via NVA (172.22.2.4).

![Option 3 Verify](/Images/3-Validation.png)















