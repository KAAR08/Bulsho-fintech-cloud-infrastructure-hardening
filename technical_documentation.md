# From Zero to Zero Trust
According to the given scenario, the fintech startup is cloud-based, hosting its API on an Ubuntu Server and its database within a PostgreSQL VM on Azure. To harden the system and mitigate the risk of a data breach, a multi-layered defense-in-depth architecture with complementary security solutions should be engineered and deployed. Because it is neither explicitly stated nor implied that the startup maintains any on-premises assets, this strategy will focus strictly on cloud network, system, and application-level security.

## Network Segmentation and Zoning
Hosting both the API server and the database on the same machine represents a poor security practice. If a malicious attacker compromises the application running on the server, they can easily move laterally and gain full access to the database contents.

To prevent such incidents, network segmentation and zoning are highly recommended. Because the API server interacts with the general public, it should be placed within a Demilitarized Zone (Cloud-DMZ). Conversely, the database must be completely restricted from public access and configured to communicate solely with the API server. Therefore, the database should be isolated within a private subnet (Restricted Zone) with no public IP address.

Additionally, using Network Security Groups (NSGs), firewall rules should be configured to allow inbound public traffic to the DMZ only on port 443 (HTTPS). Similarly, a rule must be established on the Restricted Zone that permits inbound traffic exclusively from the DMZ subnet on port 5432 (the default PostgreSQL port) and on port 22 for database server administration by the startup system admin through multihop ssh tunneling.

Below is the network design architecture:
<img src="/screenshots/network_architecture.png" alt="Zonned Network Architecture">

### Creating Virtual Network
To establish an isolated, multi-layered defensive environment, i begin by provisioning a dedicated Virtual Network (VNET) within Microsoft Azure. This network serves as the foundational boundary for the segmented architecture.

A new resource group named <i>rg-bulsho-fintech-prod</i> is created to logically group and manage all production assets for the startup.

This VNET is deployed within Africa, specifically South Africa (My Azure plan showed South Africa as the nearest center) to ensure regional digital compliance and low latency for regional infrastructure, as most service users are in Nairobi and the startup plans to expand to other regions in the continent.
<img src="/screenshots/VNET_config.png" alt="VNET Bascic Configuration">

I configured the private ip address and range for the startup VNET. /24 subnet mask was intentionally selected based on the startup's current size and relatively small user base. The scope was limited to 256 addresses to prevent massive unallocated addresses from sitting idle.
<img src="/screenshots/VNET_ip_config.png" alt="VNET Address Space Configuration">


### Restricted Zone (Private Subnet) Configuration
The first subnet created is dedicated strictly to the database layer's isolated subnet, referred as Restricted Zone/Private Subnet in the architecture.The subnet is allocated a /26 subnet mask, establishing an IP address range of 172.16.0.0 - 172.16.0.63, this provides 64 discrete IP addresses, which yields ample address space for database without consuming the entire parent network.
<img src="/screenshots/private_subnet_config1.png" alt="Private Subnet Address Space">

I checked the "Enable private subnet" checkbox to cut off default outbound internet access for any machines in this subnet. It protects the database VM from unauthorized data exfiltration attempts and prevents downloading files directly from the web.

As visually stated in the architecture, machines in this subnet are meant to be entirely isolated. Therefore, to glue this, "NAT gateway" is set to None.

Additionally, i created a new Network Security Group (NSG) named snet-private-nsg to provide stateful firewall mechanism needed to enforce zoning rules. This allows writing of explicit rule that drops all incoming traffic except for legitimate PostgreSQL traffic (on port 5432) and SSH (on port 22) originating specifically from our API server's DMZ subnet.

<img src="/screenshots/private_subnet_sec_config.png" alt="Private Subnet Sec Config">



### DMZ-Cloud Zone (Public Subnet) Configuration
This subnet acts as our Cloud-DMZ, housing the public interactive endpoints while keeping them separate from the database infrastructure.
Azure automatically segments the network space linearly. Since our database subnet (snet-db-private) claimed the first block from 172.16.0.0 to 172.16.0.63, this subnet begins precisely at the next available contiguous address, ensuring no overlapping spaces.
The subnet has 64 addresses for ample scaling out of API servers due to anticipated user growth.
<img src="/screenshots/public_subnet_config.png" alt="Public Subnet Basic Config">

I left "Enable private subnet (no default outbound access)" unchecked to allow the workloads and VMs in this subnet to have internet connectivity. This is crucial for the Ubuntu Server as it needs to perform outbound tasks such as resolving external API dependencies and communicating with external fintech payment gateways. I also enforced a separate NSG for this subnet to ensure complete separation of duties.
<img src="/screenshots/public_subnet_config1.png" alt="Public Subnet Sec Config">

As shown below, the zones are now set.
<img src="/screenshots/vnet_subnets.png" alt="Vnet zones">

After confirming that there is no conflicting IP space and incorrect configurations, i deployed the VNET.
<img src="/screenshots/vnet_deployment.png" alt="Vnet zones">

