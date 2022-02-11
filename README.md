# Azure Virtual Network Architectures with 3rd party NVA
## Scenario
Azure customers have varied security requirements. A common pattern for some is to have traffic between VNET and OnPrem be inspected by a firewall in Azure, while have VNET to VNET traffic bypass any such inspection. There are several design options for this security pattern when the firewall is a 3rd Party Network Virtual Appliance (NVA).
## Option 1: Hub-Spoke with NVA in Hub, with full mesh among Spoke VNETs
Hub-Spoke with NVA in Hub is well understood and clearly documented in the [Cloud Adoption Framework](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/hub-spoke-network-topology) and in the [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/dmz/nva-ha). The slight difference in this option is all VNETs are also directly peered amongst each other to allow direct communication without passing through firewall. 

For VNET to OnPrem traffic, User Defined Routes (UDRs) are required at the Spoke VNETs to ensure traffic is forwarded to the NVA. In order to ensure symmetry for return traffic across stateful firewalls, UDRs are required at the GatewaySubnet pointing to the NVA (or the ILB front-ending the NVA in an HA configuration) for the Spoke VNET prefix destinations.

The topology looks as follows:

![Option 1](/Images/Option1)

The advantage of this option is it is easy to understand. The disadvantage is the roughly n^2 number of peers which are required to build the full mesh.
## Option 2: Virtual WAN for direct communication between VNETs, and classic Hub to OnPrem
With this option, the Spoke VNETs are connected to both a Virtual WAN Hub as well as a classic Hub hosting a Virtual Network Gatweay (VNG). The Virtual WAN connectivity replaces the full mesh from the previous option. Communication to on-prem is facilitated via the classic Hub.

One key point to understand specific to this option is an Azure VNET can use only one gateway to learn remote routes. The gateway can be either local to the VNET or via a remote peer. Since VWAN has a “gateway” service, specifically the route service gateway, VWAN becomes the Spoke VNETs’ (remote) gateway. This implies the Spoke’s to the classic Hub cannot use the “Use Remote Gateway” option on the peer link because the Spoke already has a gateway (in VWAN). 

The VNETs will learn about each other’s prefixes and will communicate to each other via the vHubs. To reach on prem, UDRs are defined at the Spoke VNET subnets for the aggregate prefix representing onprem pointing to the NVA (or an ILB front-ending the NVA in an HA configuration). The NVA will analyze the traffic against the firewalling rules and then forward the traffic out the appropriate vNIC based on the NVA rules. The vNIC resides in the classic Hub and is aware of the OnPrem routes through the VNG. As in previous option, in order to ensure symmetry for return traffic, UDRs are required at the GatewaySubnet pointing to the NVA (or the ILB front-ending the NVA in an HA configuration).

In addition, a Route Server in the Classic Hub is required for OnPrem to learn the Spoke routes. If the NVA speaks BGP, it can originate a supernet of all the Spokes. This supernet aggregate will be reflected by the Route Server down to OnPrem. This supernet is purely defined and propagated for the purposes of ensuring traffic does not black hole, especially when the OnPrem connectivity is ExpressRoute. (If it is VPN, it is possible to configure static routing on both sides.)

On the VWAN side, all the connections associate and propagate to the Default route table or default label in order to simplify the configuration. 

The topology looks as follows:

![Option 2](/Images/Option2)

The advantage of this option is there is separation of Hub functions: a vHub for a simple any-to-any configuration, and a “DMZ” Classic Hub for traffic to OnPrem. Furthermore, this option eliminates the n^2 peering from the previous option. The disadvantage is the introduction of a new function Azure Route Server, and the NVA would need to enable BGP to peer with ARS and originate the supernet.

## Option 3: Virtual WAN Custom Route Tables
In this option, Virtual WAN Custom Route Tables are used. It is documented [here](https://docs.microsoft.com/en-us/azure/virtual-wan/scenario-route-through-nvas-custom#alternate). 

In the Virtual Hub Route Tables need to be added to the predefined Default Route Table: DMZ Route Table, and VNET Route Table. 

