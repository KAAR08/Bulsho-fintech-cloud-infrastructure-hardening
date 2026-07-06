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

## Virtual Machines Provisioning

The API server and database should sit on servers. These servers are created as Virtual Machines.

### API Server VM Provisioning

I chose to go with Ubuntu Server 22.04 LTS - Gen 2 for the startup's API server because it ensures five years of predictable, enterprise-grade security patches and system stability. Gen 2 machine also offers critical cloud security features that protect the OS against rootkits and boot-level malware.
<img src="/screenshots/api_server.png" alt="APi Server">
<img src="/screenshots/api_server1.png" alt="APi Server">
<img src="/screenshots/api_server2.png" alt="APi Server">

I mapped the api-server directly to our frontline network fabric during the provisioning phase. I selected our vnet-bulsho-fintech virtual network and placed the instance within the public-facing snet-api-public subnet (172.16.0.64/26). To ensure external consumers can seamlessly interact with the application, I enabled a new public IP resource designated as api-server-ip. Additionally, I explicitly set the NIC network security group to None since the subnet is already bound to snet-public-nsg, ensuring a clean design that handles all firewall rules at the DMZ boundary.
<img src="/screenshots/api_server3.png" alt="APi Server">
<img src="/screenshots/api_server4.png" alt="APi Server">

Once the deployment process wrapped up, I verified the live status and operational specifications of the compute resource. The api-server is now successfully running in Switzerland North (Zone 1) under the production resource group, rg-bulsho-fintech-prod. It has been provisioned with a Standard D2s v3 size featuring 2 vCPUs and 8 GiB of memory, and is actively listening on its assigned primary public IP address, giving the fintech API a solid and scalable environment to begin handling traffic.

<img src="/screenshots/api_server5.png" alt="APi Server">
<img src="/screenshots/api_server6.png" alt="APi Server">

### Database Server VM Provisioning

Bulsho fintech’s app is powered by PostgreSQL database. Bulsho fintech’s priority is achieving granular control over the environment and remaining highly cost-sensitive. With this in mind, the most appropriate choice is to provision Ubuntu VM and install database services (in later phase).

I provisioned a similar virtual machine as the API server and placed it in the private subnet.
<img src="/screenshots/db_server.png" alt="DB Server">

## Network Hardening

Now, it is time to harden each subnet by creating rules on the Network Security Groups (NSGs) to explicitly define which categories of packets to allow and which to deny.

## Hardening the Public Subnet (Cloud-DMZ)

This is where the API server sits. The host runs our public-facing APIs along with a pgAdmin web interface for database interaction and management. Additionally, to facilitate secure file transfers and remote management, SSH is enabled. With this in mind, I created three inbound security rules on the public subnet NSG:

- Allow SSH traffic from the administrator's specific IP address to the API server on port 22.
- Allow inbound HTTPS packets on port 443 from any source IP address to accommodate general public user traffic.
- Allow HTTP packets from the administrator's IP address to port 80 for restricted web-based management access.
  <img src="/screenshots/nsg_rules_public_subnet.png" alt="Public Subnet NSG Rules">

### Validating the NSG rules and testing connections
- Rule 110: Connecting the server from admin’s computer using SSH

To verify that my Network Security Group rules are functioning correctly, I initiated an SSH connection from my local management machine. By executing the command <i>ssh -i ./Downloads/api-server_key.pem <username>@<PUBLIC_IP></i>, I authenticated securely using the pre-configured private key file rather than a password.
<img src="/screenshots/ssh_connection_successful.png" alt="NSG Rule Validation">

As shown above, the connection succeeded perfectly. The Ubuntu banner confirms a successful login to the api-server host. Running the commands whoami and pwd further validates that I have secure shell access.

To confirm, that only ssh connections initiated from local management machine (admin machine) goes through the file, i tried to initiate from a different machine (just changed the IP by connecting to VPN) using the correct authentication keys. The connection failed with "connection reset by peer by server" message.
<img src="/screenshots/failed_ssh.png" alt="Failed SSH to server">


- Rule 120:  HTTPS traffic on port 443

This was accessible from any device with internet connection as the NSG allows traffic indiscriminate of IP address.
<img src="/screenshots/successful_443.png" alt="Successful Traffic on Port 443">


- Rule 130: HTTP connection from admin’s machine (local management machine) to api server.

I tested the HTTP access rule by navigating by seeting up simple web page on the server and navigating to the server's public IP address from my local management browser. The connection went through successfully because the NSG is configured to allow traffic on port 80 from the admin machine. I will use this for pg4admin to manage database graphically.

<img src="/screenshots/traffic_from_admin_to+80.png" alt="Traffic from local management Machine to port 80 server">

The server log shows / route access by local management machine
<img src="/screenshots/server_log.png" alt="Server log">

I checked if port 80 is accessible from another machine, the site was unreachable.
<img src="/screenshots/failed_port_80.png" alt="Failed Port 80 access">

The unauthorized IP request was successfully blocked by Network Security Group (NSG), hence failed to hit the api server.


## Hardening the Private Subnet (Restricted Zone)
This subnet is created to remain hidden and inaccessible from the public. The subnet houses the database server, which is a critical asset for Bulsho Fintech. The database can only be communicated with by the API server by virtue of retrieving information from and writing to the database. Albeit, it is deemed necessary for remote administration of this database. To manage the database, I decided to use the API server as a multi-hop tunnel. I could’ve used Azure Bastion, but to be mindful of Bulsho Fintech’s cost sensitivity, I opted to use the API server as the linkage between the admin’s machine and the database server. That said, the private subnet permits the following categories of packets from the API server’s private IP:

- Communication on port 22 for SSH and remote server management.

- Communication on port 5432 for database interaction by pgAdmin.
<img src="/screenshots/private_subnet_nsg_rules.png" alt="Private Subnet NSG Rules">

