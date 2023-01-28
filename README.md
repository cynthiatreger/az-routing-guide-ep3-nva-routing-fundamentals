# Episode #3: NVA Routing: What if a VM hosts an OS with routing capabilities?

An Azure VM running a 3rd Party image with networking capabilities is referred to as an [NVA](https://azure.microsoft.com/en-us/blog/best-practices-to-consider-before-deploying-a-network-virtual-appliance/) (Network Virtual Appliance). An NVA could be a firewall, a router, a proxy, an SDWAN endpoint, an IPSec concentrator etc.

To illustrate the NVA use case, a Cisco CSR is deployed in the NVAsubnet of the hub VNET. In our scenario, this NVA terminates connectivity to On-Prem branches.

The On-Prem connectivity via the Virtual Network GW has been removed to avoid unnecessary complexity for now.


