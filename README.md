# Azure Virtual Network Architectures with 3rd party NVA
## Scenario
Azure customers have varied security requirements. A common pattern for some is to have traffic between Virtual Networks (VNETs) and OnPrem be inspected by a firewall in Azure, while have VNET to VNET traffic bypass any such inspection. There are several design options for this security pattern when the firewall is a 3rd Party Network Virtual Appliance (NVA).
## Option 1: Hub-Spoke with NVA in Hub, with full mesh among Spoke VNETs
The Hub-Spoke Design with NVA in Hub is well understood and clearly documented in the [Cloud Adoption Framework](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/hub-spoke-network-topology) and in the [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/dmz/nva-ha). The slight difference in this option is all VNETs are also directly peered amongst each other to allow direct communication without passing through firewall. 

For VNET to OnPrem traffic, User Defined Routes (UDRs) are required at the Spoke VNETs to ensure traffic is forwarded to the NVA. In order to ensure symmetry for return traffic across stateful firewalls, UDRs are required at the GatewaySubnet pointing to the NVA (or the ILB front-ending the NVA in an HA configuration) for the Spoke VNET prefix destinations.

The topology looks as follows:

![Option 1](/Images/Option1.png)

The advantage of this option is it is easy to understand. The disadvantage is the roughly n^2 number of peers which are required to build the full mesh.
## Option 2: Virtual WAN for direct communication between VNETs, and Classic Hub to OnPrem
With this option, the Spoke VNETs are connected to both a Virtual WAN Hub as well as a Classic Hub hosting a Virtual Network Gateway (VNG). The Virtual WAN connectivity replaces the full mesh from the previous option. Communication to on-prem is facilitated via the Classic Hub.

One important point to understand is an Azure VNET can use only one gateway to learn remote routes. The gateway can be either local to the VNET or via a remote peer. Since VWAN has a “gateway” service, specifically the route service gateway, VWAN becomes the Spoke VNETs’ (remote) gateway. This implies the Spoke’s to the Classic Hub cannot use the “Use Remote Gateway” option on the peer link because the Spoke already has a gateway (in VWAN). 

The VNETs will learn about each other’s prefixes and will communicate to each other via the vHubs. To reach on prem, UDRs are defined at the Spoke VNET subnets for the aggregate prefix representing onprem pointing to the NVA (or an ILB front-ending the NVA in an HA configuration). The NVA will analyze the traffic against the firewalling rules and then forward the traffic out the appropriate vNIC based on the NVA rules. The vNIC resides in the Classic Hub and is aware of the OnPrem routes through the VNG. As in previous option, in order to ensure symmetry for return traffic, UDRs are required at the GatewaySubnet pointing to the NVA (or the ILB front-ending the NVA in an HA configuration).

In addition, a Route Server in the Classic Hub is required for OnPrem to learn the Spoke routes. If the NVA speaks BGP, it can originate a supernet of all the Spokes. This supernet aggregate will be reflected by the Route Server down to OnPrem. This supernet is purely defined and propagated for the purposes of ensuring traffic does not black hole, especially when the OnPrem connectivity is ExpressRoute. (If it is VPN, it is possible to configure static routing on both sides.)

On the VWAN side, all the connections associate and propagate to the Default route table or default label in order to simplify the configuration. 

The topology looks as follows:

![Option 2](/Images/Option2.png)

The advantage of this option is there is separation of Hub functions: a vHub for a simple any-to-any configuration, and a “DMZ” Classic Hub for traffic to OnPrem. Furthermore, this option eliminates the n^2 peering from the previous option. The disadvantage is the introduction of a new function Azure Route Server, and the NVA would need to enable BGP to peer with ARS and originate the supernet.

## Option 3: Virtual WAN Custom Route Tables
In this option, Virtual WAN Custom Route Tables are used as described [here](https://docs.microsoft.com/en-us/azure/virtual-wan/scenario-route-through-nvas-custom#alternate). The vHub has not only peering Connections built to the Spoke VNETs, but it also has peering Connections built to a DMZ VNET. The 3rd party NVA resides in the DMZ VNET.

In vHub, a “Default” and a “None” route table are predefined. For this option, two additional Route Tables are created: the DMZ Route Table (RT-DMZ) and VNET Route Table (RT-VNET). By configuring appropriate Propagation and Association on the Connections, the VNETs do not learn the OnPrem prefixes directly, but only via defined Static Routes pointing to the DMZ Connection with the NVA (or ILB front-ending NVAs) as Next Hop. Similarly, OnPrem reaches the Spoke VNET via a defined Static Route pointing to the DMZ Connection with the NVA (or ILB front-ending NVAs) as Next Hop. 

The DMZ, however, is learning both OnPrem and Spoke VNET prefixes and acts as the bridge between VNET and OnPrem, hence satisfying the requirement of filtering traffic between VNET to OnPrem.

The Association on a Connection may be viewed as the VNET (Connection) will learn, and Propagation on a Connection may be thought of as what the VNET (Connection) will advertise its existence to. For example, if Spoke-VNET1 Connection is propagating to RT-DMZ Route Table, that means anything associated with RT-DMZ will learn Spoke-VNET1’s prefix. Similarly, if Spoke-VNET1 is associated with RT-VNET, it will learn of the prefixes that are propagating to RT-VNET.

This Option may be illustrated as follows:

![Option 3](/Images/Option3.png)

The Assocation and Propagation are defined as follows:
Connection	  Association	  Propagation
VNET-Spoke	  RT-VNET	      RT-VNET, RT-DMZ
DMZ	          RT-DMZ	      RT-VNET, Default
Branch	      Default     	Default, RT-DMZ

Static Route definitions: 
In RT-VNET route table -> Destination Prefix Branch -> Next Hop Connection-DMZ -> Next Hop IP -> NVA
In Default route table -> Destination VNET Supernet -> Next Hop Connecction-DMZ -> Next Hop IP -> NVA


## Appendix A: Testing the Options
The above options were validated in simplified lab scenario (using VPN gateway). Below are some tips to get started:

### Option 1: Hub-Spoke with NVA in Hub, with full mesh among Spoke VNETs
This is a well documented scenario. Refer to the documentation in [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/dmz/nva-ha).

### Option 2: Virtual WAN for direct communication between VNETs, and Classic Hub to OnPrem
