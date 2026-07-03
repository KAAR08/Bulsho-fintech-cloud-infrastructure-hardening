# From Zero to Zero Trust
According to the given scenario, the fintech startup is cloud-based, hosting its API on an Ubuntu Server and its database within a PostgreSQL VM on Azure. To harden the system and mitigate the risk of a data breach, a multi-layered defense-in-depth architecture with complementary security solutions should be engineered and deployed. Because it is neither explicitly stated nor implied that the startup maintains any on-premises assets, this strategy will focus strictly on cloud network, system, and application-level security.

## Network Segmentation and Zoning
Hosting both the API server and the database on the same machine represents a poor security practice. If a malicious attacker compromises the application running on the server, they can easily move laterally and gain full access to the database contents.

To prevent such incidents, network segmentation and zoning are highly recommended. Because the API server interacts with the general public, it should be placed within a Demilitarized Zone (Cloud-DMZ). Conversely, the database must be completely restricted from public access and configured to communicate solely with the API server. Therefore, the database should be isolated within a private subnet (Restricted Zone) with no public IP address.

Additionally, using Network Security Groups (NSGs), firewall rules should be configured to allow inbound public traffic to the DMZ only on port 443 (HTTPS). Similarly, a rule must be established on the Restricted Zone that permits inbound traffic exclusively from the DMZ subnet on port 5432 (the default PostgreSQL port) and on port 22 for database server administration by the startup system admin through multihop ssh tunneling.

Below is the network design architecture:
<img src="/screenshots/network_architecture.png" alt="Zonned Network Architecture">


