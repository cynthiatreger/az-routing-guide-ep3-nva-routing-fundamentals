# Episode #3: NVA Routing: What if a VM hosts an OS with routing capabilities?

An Azure VM running a 3rd Party image with networking capabilities is referred to as an [NVA](https://azure.microsoft.com/en-us/blog/best-practices-to-consider-before-deploying-a-network-virtual-appliance/) (Network Virtual Appliance). An NVA could be a firewall, a router, a proxy, an SDWAN hub, an IPSec concentrator etc.

To illustrate the NVA use case, a Cisco CSR is deployed in the NVAsubnet (10.0.10.0/24) of the hub VNET. In our scenario this NVA will terminate connectivity to On-Prem branches (SDWAN or IPSec for example).

The On-Prem connectivity represented in [Episode #1](https://github.com/cynthiatreger/az-routing-guide-ep1-vnet-peering-and-virtual-network-gateways) and [Episode #2](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing) via the Virtual Network GW has been removed to avoid unnecessary complexity for now.

<img width="550" alt="image" src="https://user-images.githubusercontent.com/110976272/215292829-2d9ba955-711a-4d96-8666-9feb5c312bb8.png">

# 3.1. NVA default routing configuration & packet walk
Let's start without any branches connected to the Concentrator NVA and observe its default configuration.

<img width="997" alt="image" src="https://user-images.githubusercontent.com/110976272/215293529-596345e5-43cf-4637-9ee6-4dfe96faffd1.png">

## 3.1.1. NVA *Effective routes* & NVA routing table alignemnt
When freshly deployed, the CSR comes with a default configuration, inherited from the VM on which it is hosted.

Interface GigabitEthernet1 of the router has retrieved the IP address of the VM’s NIC:

<img width="623" alt="image" src="https://user-images.githubusercontent.com/110976272/215293632-9da2475d-d15f-41d3-ae12-2c02e035cdce.png">

Let’s look at the routing table (OS-level):

<img width="441" alt="image" src="https://user-images.githubusercontent.com/110976272/215293719-44f074c9-4e2b-4620-a4fb-5bdaf24c58d8.png">

As expected from a router, it contains the directly connected route of its GigabitEthernet1 interface (10.0.10.0/24). 

And since by default all resources in a VNET can communicate outbound to the internet, the 0.0.0.0/0 route of the NIC *Effective routes* is reflected in the Concentrator routing table, pointing towards the default GW of the subnet (10.0.10.1*).

\*[In Azure, 5 IPs are reserved per subnet](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq#are-there-any-restrictions-on-using-ip-addresses-within-these-subnets:~:text=For%20example%2C%20the,Network%20broadcast%20address.): 1 network + 1 broadcast + 3 Azure reserved IPs (1 subnet default gateway + 2 DNS)

[168.63.129.16](https://learn.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16) and [169.254.169.254](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/instance-metadata-service?tabs=windows) are also part of the default configuration of any device in Azure and are used for the communication to Azure management platform resources. These IPs re not relevant in the context of this article.

Because the router’s OS route processing differs from standard VNET routing, there isn’t an exact 1:1 match between the entries of the NVA routing table (containing the local subnet + a default route) and the (underlying) VM’s NIC *Effective routes* (VNET ranges + default route). 

The data plane reachability between the different VNETs is validated with pings:

<img width="500" alt="image" src="https://user-images.githubusercontent.com/110976272/215294234-ba8cdc6d-e439-48bc-b4e5-f4540030eb39.png">

## 3.1.2.	Packet walk
Let’s take the example of the ping from the Concentrator NVA (10.0.10.4) to Spoke1VM (10.1.1.4 in the Spoke1 peered VNET).

1. Packets are first processed by the router OS.
Because in the CSR routing table there isn’t any match for the Spoke1VM subnet (10.1.1.0/24) or for the Spoke1 VNET range (10.1.0.0/16), packets will be caught by the default route pointing towards 10.0.10.1 (the VNET default GW). 
2. As 10.0.10.1 belongs to the 10.0.10.0/24 subnet for which an outbound interface is known in the routing table (GigabitEthernet1), the usual recursive routing will result in the packets being forwarded out the GigabitEthernet1 interface.

In traditional networking, the packet would be processed through an SFP (inserted in a physical port labelled “GE1”) and according to the L3 routing decision made by the router. 

3. :arrow_right: **In Azure, once forwarded to the (logical) GigabitEthernet1 interface, packets reach the underlying associated physical NIC, where they are routed based on the NIC *Effective routes* and the packet destination IP, and are finally sent to the wire accordingly** (here the destination matches the 10.1.0.0/16 entry know via VNET peering).

Understanding this additional step and most importantly the need of alignment between a VM’s Effective routes and the routing table sitting on top of the VM has been a key learning for me.

# 3.2. On-Prem connectivity via an NVA

We are now back to our initial setup, with On-Prem branches connected to the Concentrator NVA.
Loopback addresses have been configured on the CSR for branch emulation.

## 3.3.2.	NVA routing table & NVA <=> Branch reachability 

Connectivity between the Concentrator NVA and the On-Prem branches is confirmed by the Concentrator routing table and successful pings:

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/110976272/215296126-6d1982c5-871d-4c23-927e-fa55fb7a41f0.png">

## 3.3.2.	NVA *Effective routes* and NVA routing table misalignment
Although existing in the Concentrator NVA routing table, the branch prefixes are NOT reflected on the underlying VM’s NIC Effective routes nor known or reachable by any other VM in the VNET or peered VNETs, resulting in failed connectivity to the On-Prem branches.

<img width="1124" alt="image" src="https://user-images.githubusercontent.com/110976272/215295807-d7c6c83f-b744-4606-ac66-fee7aebb717a.png">

## 6.2.2.	Solution: Align the data-plane [NIC-level] to the control-plane [OS-level]
To enable connectivity with this new loopback address in the testing environment, the NICs of the VMs must also know about its IP address, having the information at the NVA OS level is not enough. 
This is achieved by 1) creating an Azure route table (or updating an existing one), 2) configuring an entry (a UDR) in this route table for the 7.7.7.7/32 destination with the next hop being a Virtual Appliance with IP address = 10.0.0.5, and 3) associating the Route Table with any subnet requiring connectivity to that loopback address

