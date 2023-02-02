### [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
##
# Episode #3: NVA Routing Fundamentals or what if a VM hosts an OS with routing capabilities?

*Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.*
##
[3.1. NVA default routing configuration & packet walk](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#31-nva-default-routing-configuration--packet-walk)

&emsp;[3.1.1. NVA *Effective routes* & NVA routing table alignment](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#311-nva-effective-routes--nva-routing-table-alignment)

&emsp;[3.1.2.	Packet walk](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#312packet-walk)

[3.2. On-Prem connectivity via an NVA](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#32-on-prem-connectivity-via-an-nva)

&emsp;[3.2.1.	NVA routing table & reachability between the NVA and the branches](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#321nva-routing-table--reachability-between-the-nva-and-the-branches)

&emsp;[3.2.2.	Azure VM *Effective routes* and NVA routing table misalignment](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#322azure-vm-effective-routes-and-nva-routing-table-misalignment)

&emsp;[3.2.3.	Solution: Align the data-plane (VM *Effective routes*) to the control-plane (NVA routing table)](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#323solution-align-the-data-plane-effective-routes-to-the-control-plane-nva-routing-table)
##
An Azure VM running a non-native 3rd Party image with networking capabilities is referred to as an [NVA](https://azure.microsoft.com/en-us/blog/best-practices-to-consider-before-deploying-a-network-virtual-appliance/) (Network Virtual Appliance). An NVA could be a firewall, a router, a proxy, an SDWAN hub, an IPSec concentrator etc.

To illustrate the NVA use case, a Cisco CSR is deployed in a new subnet of the hub VNET ("NVAsubnet": 10.0.10.0/24), and terminates connectivity to On-Prem branches (could be SDWAN or IPSec).

This scenario represents an alternative to the On-Prem connectivity via the Azure native Virtual Network GW discussed in [Episode #1](https://github.com/cynthiatreger/az-routing-guide-ep1-vnet-peering-and-virtual-network-gateways) and [Episode #2](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing).

<img width="550" alt="image" src="https://user-images.githubusercontent.com/110976272/215292829-2d9ba955-711a-4d96-8666-9feb5c312bb8.png">

# 3.1. NVA default routing configuration & packet walk
Let's start without any branches connected to the Concentrator NVA and observe its default configuration and behaviour.

<img width="997" alt="image" src="https://user-images.githubusercontent.com/110976272/215293529-596345e5-43cf-4637-9ee6-4dfe96faffd1.png">

## 3.1.1. NVA *Effective routes* & NVA routing table alignment
When freshly deployed, the CSR comes with a default configuration, inherited from the VM on which it is hosted.

Interface GigabitEthernet1 of the router has retrieved the IP address of the NVA’s NIC:

<img width="623" alt="image" src="https://user-images.githubusercontent.com/110976272/215293632-9da2475d-d15f-41d3-ae12-2c02e035cdce.png">

Extract of the routing table (OS-level):

<img width="441" alt="image" src="https://user-images.githubusercontent.com/110976272/215293719-44f074c9-4e2b-4620-a4fb-5bdaf24c58d8.png">

As expected from a router, it contains the directly connected route of its GigabitEthernet1 interface (10.0.10.0/24). 

And since by default all resources in a VNET can communicate outbound to the internet, the 0.0.0.0/0 route of the NIC *Effective routes* is reflected in the Concentrator routing table, pointing towards the default GW of the subnet (10.0.10.1 *).

\* *[In Azure, 5 IPs are reserved per subnet](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq#are-there-any-restrictions-on-using-ip-addresses-within-these-subnets:~:text=For%20example%2C%20the,Network%20broadcast%20address.): 1 network + 1 broadcast + 3 Azure reserved IPs (1 subnet default gateway + 2 DNS).*

[168.63.129.16](https://learn.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16) and [169.254.169.254](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/instance-metadata-service?tabs=windows) are also part of the default configuration of any device in Azure and are used for the communication to Azure management platform resources. These IPs are not relevant in the context of this article.

Because the router’s OS route processing differs from standard VNET routing, there isn’t an exact 1:1 match between the entries of the NVA routing table (containing the local subnet + a default route) and the (underlying) VM’s NIC *Effective routes* (VNET ranges + default route). 

The data plane reachability between the different VNETs is validated with pings:

<img width="500" alt="image" src="https://user-images.githubusercontent.com/110976272/215294234-ba8cdc6d-e439-48bc-b4e5-f4540030eb39.png">

## 3.1.2.	Packet walk
Let’s use the example of the ping from the Concentrator NVA (10.0.10.4) to Spoke1VM (10.1.1.4 in the Spoke1 peered VNET).

1. Packets are first processed by the router OS.
Because in the CSR routing table there isn’t any match for the Spoke1VM subnet (10.1.1.0/24) or for the Spoke1 VNET range (10.1.0.0/16), packets will be caught by the default route pointing towards 10.0.10.1 (the VNET default GW). 
2. As 10.0.10.1 belongs to the 10.0.10.0/24 subnet for which an outbound interface is known in the routing table (GigabitEthernet1), the usual recursive routing will result in the packets being forwarded out the GigabitEthernet1 interface.

In traditional networking, the packet would be processed through an SFP (inserted in a physical port labelled “GE1”) and according to the L3 routing decision made by the router. 

3. :arrow_right: **In Azure, once forwarded to the (logical) GigabitEthernet1 interface, packets reach the underlying associated physical NIC, where they are routed based on the NIC *Effective routes* and the packet destination IP, and are finally sent to the wire accordingly** (here the destination matches the 10.1.0.0/16 entry know via VNET peering).

Understanding this additional step and most importantly the need of alignment between a VM’s Effective routes and the routing table sitting on top of the VM has been a key learning for me.

# 3.2. On-Prem connectivity via an NVA

We are now back to our initial setup, with On-Prem branches connected to the Concentrator NVA.

<img width="550" alt="image" src="https://user-images.githubusercontent.com/110976272/215292829-2d9ba955-711a-4d96-8666-9feb5c312bb8.png">

In real-life scenarios the On-Prem prefixes would likely be learnt dynamically by the Concentrator, via BGP over IPSec tunnels or via an SDWAN overlay for example, and the Concentrator would in return advertise the Azure IP ranges to the branches.

Whether IPSec or SDWAN, the solution run between the Concentrator and its branches is not relevant to understand the Azure routing principles. Loopback addresses have been configured on the CSR for branch emulation.

## 3.2.1.	NVA routing table & reachability between the NVA and the branches

Connectivity between the Concentrator NVA and the On-Prem branches is confirmed by the Concentrator routing table and successful pings:

<img width="1058" alt="image" src="https://user-images.githubusercontent.com/110976272/215522509-ffb8ad9c-772d-4b56-b1ea-7e431ea79aa0.png">

The static 10/8 supernet route covering the Azure environment and pointing to the subnet default gateway (10.0.10.1) is configured on the Concentrator NVA to be further advertised to the branches.

## 3.2.2.	Azure VM *Effective routes* and NVA routing table misalignment
Although existing in the Concentrator NVA routing table, the branch prefixes are NOT reflected on the underlying NVA’s NIC *Effective routes* nor known or reachable by any other VM in the VNET or peered VNETs, resulting in failed connectivity to the On-Prem branches. Step 3 of the above packet walk cannot be completed.

<img width="1151" alt="image" src="https://user-images.githubusercontent.com/110976272/215525127-66412d8b-0e17-4528-9f8f-39b11e82065c.png">

## 3.2.3.	Solution: Align the data-plane (*Effective routes*) to the control-plane (NVA routing table)
To enable end-to-end connectivity, the NICs of the VMs must also know about the branch prefixes, having the information at the NVA OS level is not enough. 

This is achieved by 1) creating an Azure *Route table* (or updating an existing one), 2) configuring a static entry (a UDR) in this *Route table* for the branch prefixes (the 192.168.0.0/16 supernet is enough) with the NVA as Next-Hop (10.0.10.4), and 3) associating the *Route table* with any subnet requiring connectivity to these branches. Steps 2 and 3 are represented below:

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/110976272/215296529-0ba577fb-0762-442b-a59e-52ad31e6377b.png">

:warning: Do make sure to follow the blue box note of enabling IP forwarding on the Concentrator NVA’s NIC. When a UDR is configured with an NVA as Next-Hop, IP forwarding must be enabled on the NVA NIC receiving the traffic or packets will be dropped.

All the subnets that should know about the On-Prem prefixes must be explicitly associated to this *Route table*, containing the /16 static route to the Concentrator NVA. For this scenario we have associated all the subnets of Spoke1 and Spoke2 VNETs and the Hub Test subnet to a *Route table* named "SpokeRT":

<img width="678" alt="image" src="https://user-images.githubusercontent.com/110976272/215543830-32e9bb56-6cae-4e6a-928c-5765bec86a4b.png">

As observed on the previous diagram, the Concentrator NVA itself doesn’t have the On-Prem prefixes in its *Effective routes*. Its subnet must therefore also be associated to a *Route table* with the 192.16.0.0/16 UDR.

:arrow_right: Although the same UDR (192.16.0.0/16 => 10.0.10.4) will be configured for the NVA and for the Spokes it is common practice to create a Route table dedicated to the subnet of the NVA, to allow for further distinct updates usually required between the Spokes and the NVA.

Here is a summary of the Concentrator NVA's *Route table* (named "ConcentratorRT"):

<img width="688" alt="image" src="https://user-images.githubusercontent.com/110976272/215297079-9faf1ef6-b1d3-477c-b612-0d49563af5e1.png">

The impact on the VMs *Effective routes* of associating these 2 *Route tables* to the Spoke subnets and the NVA subnet respectively is represented on the updated diagram below:

<img width="1173" alt="image" src="https://user-images.githubusercontent.com/110976272/215555521-bdce8934-c420-4fab-8159-3e602aec031a.png">

:arrow_right: The branch prefixes are not propagated by the Concentrator NVA in its VNET and peered VNETs like a Virtual Network Gateway would do. Connectivity is achieved via static routes pointing to the Concentrator and configured on every target subnet.

:arrow_right: The *Gateway Transit* and *Gateway route propagation* settings only apply to native Azure gateways (Expressroute or VPN Virtual Network Gateways, or Azure Route Servers as we will see in Episode #5) and as a result are not relevant in the context of the current use case (UDRs + non-native Concentrator NVA).
##
### [>> to EPISODE #4](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas) coming out soon
