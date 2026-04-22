# HyperV-ADDS-DNS-DHCP-Redundancy-Lab
In this lab, I configure three Windows Server 2022 machines using Hyper-V: one as a primary AD DS, DNS, and DHCP server; one as a secondary domain controller with DNS; and one as a backup DHCP server to provide redundancy and high availability. I use a Hyper-V private virtual switch for this setup because it creates a fully isolated virtual network. In this environment, all virtual machines can communicate with each other, allowing me to build a complete, interconnected lab entirely within the host. At the same time, there’s no connection to my local area network, which means I can safely test configurations and simulate real-world scenarios without risk of affecting my live environment.

## Hyper-V
Hyper-V is built into Windows as an optional feature for Windows Server 2025 -2016, Windows 10 (Pro or Enterprise), or Windows 11 (Pro or Enterprise); thus, there is no Hyper-V download. To enable Hyper-V on Windows 10 Pro, click Start and type "Turn Windows features on or off". Then check Hyper-V and hit OK. Once the installation is done, the computer will be restarted.

![HyperV](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/5c47b2efc04972cd931b51477659634e081e6424/images/HyperV.PNG)

## Building the private switch in Hyper-V
Click Start, type “Hyper-V Manager”, and open the Hyper-V Manager console. From the dashboard, under the "Actions" tab, click on "Virtual Switch Manager". Click on Private from the list, then "Create Virtual Switch". 
![PrivateVirtualSwitch](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/5c47b2efc04972cd931b51477659634e081e6424/images/Private%20Switch%20(1).png)

Name the switch and hit Apply. Now that the private switch is ready, select it under the “Configure Networking” step when creating a new virtual machine in the New Virtual Machine Wizard.
![PrivateVirtualSwitch](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/5c47b2efc04972cd931b51477659634e081e6424/images/Private%20Switch%20(2).png)

From Hyper-V Manager, click on New, then Virtual Machine. Complete the New Virtual Machine Wizard three times to create the three Windows Server 2022 machines.

Once all virtual machines are created and running, rename them as follows and configure their static IP addresses:

**DC01 (Primary Domain Controller)**
* Static IP: 192.168.0.2
* Roles: Active Directory Domain Services (AD DS), DNS
* Purpose: Acts as the primary domain controller responsible for authentication, directory services, and DNS resolution.

**DC02 (Secondary / Backup Domain Controller)**
* Static IP: 192.168.0.3
* Role: Additional Domain Controller, DNS
* Purpose: Provides redundancy and high availability for Active Directory services, ensuring fault tolerance in case DC01 becomes unavailable.

**DHCP02 (DHCP Server)**
* Static IP: 192.168.0.4
* Role: DHCP Server
* Purpose: Manages IP address allocation within the network and is configured as a backup DHCP server to support continuity of network services.

Promote DC01 as the domain controller: From Server Manager's menu bar, click on Manage > "Add Roles and Features". "Add Roles and Features Wizard" will start. Progress it by setting up the "Active Directory Domain Services". Once the installation is completed, before closing the wizard, click on "Promote this server to a domain controller". 

In this lab, I used int.homelab.local as the domain name. Select “Add a new forest” and enter your desired domain name (in this case, int.homelab.local), then click Next and proceed through the wizard. On the Additional Options page, the wizard will automatically generate a NetBIOS name (in this case, INT). 
![NetBIOS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/8d51a836d91a6891612f33dc432ce11fad8902a4/images/NetBIOS%20(1).PNG)

Change this to HOMELAB to make it more intuitive and avoid confusion for users during login. Confirm the paths, click Next. Review the options. On the next page, the configuration wizard will check the prerequisites. Once it is done, as long as we don't see errors that are marked with red signs, we can click on the "Install" button. After the installation is completed, the server will be restarted.
![NetBIOS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/8d51a836d91a6891612f33dc432ce11fad8902a4/images/NetBIOS%20(2).PNG)

Sometimes administrators might choose to use the same namespace for both their external domain and their internal Active Directory domain (for example, itsupport.com). While this approach can work, it introduces several challenges.

When the same DNS name is used internally and externally, both internal and external DNS servers host the same zone. For instance, an external user would query a public DNS server to resolve the website, while an internal client would typically query an internal DNS server. Although this setup appears functional, it creates a significant concern: internal DNS servers may contain sensitive records (such as those used by Active Directory) that should never be exposed externally.

If these internal DNS records are accessible from the internet, attackers can perform **footprinting** to gather information about the network, including available services and system structure. Even though this is not a direct attack, it provides valuable insight that can be used for future exploitation.

This scenario often requires implementing split-brain DNS, where separate DNS views are maintained for internal and external users. However, this adds complexity to the configuration and management of the environment.

To avoid these issues, a common best practice is to use a separate internal namespace. In this lab, I followed this approach by prefixing the domain with an internal identifier. Instead of using homelab.local for both internal and external purposes, I configured the internal domain as int.homelab.local. This allows clear separation between internal and external DNS, improves security, and simplifies overall network design.

## Domain Name System (DNS) - DC01
When installing Active Directory Domain Services (AD DS), the DNS Server role is installed and configured automatically. DNS plays a critical role in Active Directory by enabling clients to locate domain controllers and other services within the domain. Without a properly functioning DNS infrastructure, core Active Directory operations such as authentication, replication, and service discovery would fail. Let’s take a quick look at what Active Directory automatically configures in DNS during deployment: Server Manager Dashboard > Tools > DNS
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(1).PNG)

DNS > DC01 > Forward Lookup Zones. You can see that the DNS Forward Lookup Zone is already created. A forward lookup zone resolves a hostname (for example, dc01) to its corresponding IP address. We can see that DC01 has been properly added to the DNS. This is also called A DNS "A record".
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/10936461f6b463523768417fb166ef2b90d1b035/images/DNS%20(2.1).PNG)

If you right-click the zone and open Properties, you can review the Start of Authority (SOA) record. This shows the authoritative name server for the zone and confirms that the DNS configuration is valid and functioning correctly. You’ll also see that the hostname has already been automatically registered, indicating that DNS is properly integrated with Active Directory.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(2).PNG)
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(3).PNG)

However, you’ll notice that a reverse lookup zone is not created by default. Without it, the DNS server cannot resolve IP addresses back to hostnames (PTR records), which can cause issues with certain services, logging, and troubleshooting.
To fix this, we can create a reverse lookup zone: Right-click Reverse Lookup Zones → New Zone.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(4).PNG)

![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(5).PNG)

Select Primary Zone, and since this is a domain environment, choose to store the zone in Active Directory.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(6).PNG)

One key difference compared to standalone DNS is the replication scope option. Here, you can choose to replicate the DNS zone to all DNS servers running on domain controllers in the domain. This ensures that when additional domain controllers are added (such as DC02), the DNS data is automatically shared—making the setup more scalable and resilient.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(7).PNG)

Next, select IPv4 Reverse Lookup Zone
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(8).PNG)

Enter the network ID 192.168.0
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(9).PNG)

Proceed with the wizard and allow secure dynamic updates. This enables domain-joined machines to automatically create and update their DNS records, which is essential for Active Directory environments.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(10).PNG)

Once the wizard is complete, the reverse lookup zone is created.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(11).PNG)

Double-click on Start of Authority under Reverse Lookup Zones. A **Start of Authority (SOA)** DNS record is a mandatory resource record that defines the beginning of a DNS zone and contains essential administrative details. These include the primary name server for the zone, the administrator’s contact information, and important timing parameters such as refresh, retry, expire, and default TTL values used for zone replication and caching.   
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(12).PNG)

A **Name Server (NS)** record identifies which servers are authoritative for a domain, acting as a directory that directs traffic to the correct web host. These records are essential for directing queries to the right IP address, typically requiring at least two servers for redundancy to prevent website downtime. The IP Address of the Name Server is unknown; let's verify it. 
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(13).PNG)

Click on Edit to edit the name server record. 
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(14).PNG)

Click on the Resolve button to validate it.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(15).PNG)

Lastly, to fix a pointer record for DC01 under the Forward Lookup Zone. Right-click on DC01 and Properties.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(16).PNG)

Check "Update associated pointer (PTR) record and Apply.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(17).png)

When you click on Refresh.
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(18).png)

The Reverse Lookup Zone for DC01 will be updated. A Pointer (PTR) record maps an IP address to a hostname, enabling "reverse DNS lookup" (IP-to-name). It is the opposite of an 'A' record (name-to-IP).
![DNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/9f9f06a27f2fe57fc05bd19ff1076a752877f2d5/images/DNS%20(19).PNG)

DNS verification has completed.

## Dynamic Host Configuration Protocol (DHCP) - DC01
To install DHCP: From Server Manager's menu bar, click on Manage > "Add Roles and Features". Next > Click on "Role-based or feature-based installation" Next > Server Selection should be DC01.int.homelab.local (our fully qualified domain name) Next > For the server role, click on DHCP and add its features.
![DHCPsetup](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/751af3f237bdefd9bad6df93c641304fe419e40a/images/DHCP%20setup%20(1).PNG)

Complete the wizard by clicking Next through the remaining steps. Once the installation is finished, you will receive a notification prompting you to complete the DHCP Configuration. Click to continue...
![DHCPsetup](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/751af3f237bdefd9bad6df93c641304fe419e40a/images/DHCP%20setup%20(2).PNG)

This will create security groups for DHCP Server Administration.
![DHCPsetup](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/751af3f237bdefd9bad6df93c641304fe419e40a/images/DHCP%20setup%20(3).PNG)

It will automatically recognize our Active Directory credentials. Next and finish the configuration.
![DHCPsetup](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/751af3f237bdefd9bad6df93c641304fe419e40a/images/DHCP%20setup%20(4).PNG)

From Server Manager's menu bar, click on Tools > DHCP to configure.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(1).PNG)

DHCP > dc01.int.homelab.local > IPv4. Right-click on IPv4 to set a new scope.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(2).PNG)

This will launch the New Scope Wizard, which is used to define a range of IP addresses that the DHCP server can lease to clients within the network.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(3).PNG)

Name the scope
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(4).PNG)

A static IP address is used to assign a permanent, unchanging network address to devices that require consistent and reliable connectivity, such as servers, printers, and VoIP systems. That is also why we assigned static IP addresses to all three servers in this lab. It simplifies network administration, supports services that depend on fixed addressing, and can improve security by enabling IP-based allowlisting. It also ensures stable connections for services that benefit from consistent addressing, avoiding potential interruptions caused by dynamically changing IPs. For this reason, DHCP scopes are configured to avoid overlapping with statically assigned IP addresses. This ensures that the DHCP server does not accidentally distribute addresses that are already reserved for critical infrastructure components.

In this case, I configure the DHCP scope to start from 192.168.0.30 and end at 192.168.0.230, providing a total of 200 available IP addresses for dynamic allocation within the network.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(5).PNG)

Click Next, as we are not configuring any exclusions or delay settings in this scope.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(6).PNG)

DHCP lease time refers to the duration for which a device is allowed to use a specific IP address assigned by a DHCP server. Once the lease expires, the device loses that IP address, its network connection is refreshed, and it must request a new address from the DHCP server. During the lease period, the IP address remains effectively “reserved” for that device, ensuring stable connectivity.

In environments such as public networks (for example, coffee shops or guest Wi-Fi), it is common to configure shorter lease times—such as 30 minutes to 1 hour—to quickly recycle IP addresses and avoid exhaustion when many transient devices connect. In this lab environment, however, the default lease duration is sufficient, as the number of connected devices will be limited and predictable.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(7).PNG)

Hit next to configure the additional parameters.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(8).PNG)

Since we are using a private virtual switch, which is a fully isolated virtual network, we will simulate a default gateway (router) for configuration purposes. In this lab, we will assign the gateway IP address as 192.168.0.1 and click on Add.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(9).PNG)

Click on next.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(1).PNG)

The wizard is now prompting for Active Directory integration, and it detects the existing domain int.homelab.local. It is asking whether this DHCP configuration should be authorized and integrated with the parent Active Directory domain. Since this DHCP server is part of the same domain infrastructure, we confirm that it should be integrated with int.homelab.local. This ensures proper authorization within Active Directory and allows the DHCP server to securely operate in the environment. It also correctly identifies that 192.168.0.2 will be used as one of the DNS server options distributed to clients. Click Next to continue.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(11).PNG)

Click Next, as we are not configuring any WINS Servers, which are used to configure NetBIOS name resolution on client computers.
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(12).PNG)

Next!
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(13).PNG)

Finish!
![DHCP](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/deb51181a066bb3a473ae191074428b2362dd162/images/DHCP%20(14).PNG)

**To test the DHCP configuration**, I quickly added a Windows 10 Pro virtual machine as a client named PC001. As expected, it does not have a static IP address assigned. When running ipconfig, the client successfully received an IP address of 192.168.0.30, which falls within the DHCP scope configured earlier (192.168.0.30–192.168.0.230). This confirms that the DHCP server is functioning correctly and distributing addresses as intended.
![DHCPtest](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c9626a3195a4983eeda052fce054c86bb42c044d/images/DHCPtest.PNG)

*You can also observe that both the DNS server and default gateway/DHCP-related configuration are currently pointing to 192.168.0.2. This is expected at this stage of the lab, as only the primary server (DC01) is active. This configuration will be updated later once redundancy is fully implemented with DC02 and DHCP02, ensuring high availability and failover capabilities across the environment.*

**DC02 (Secondary / Backup Domain Controller)**
At this stage, we have a fully functional Active Directory environment in place. We have a domain controller, DNS server, and DHCP server all working together—handling authentication, name resolution, and IP address allocation across our virtual network. However, in a real-world scenario, relying on a single domain controller creates a single point of failure. If that server goes offline, critical services such as user authentication and DNS resolution would be disrupted.

To address this, we implement redundancy. The next step is to add a second domain controller (DC02) to the environment. This provides fault tolerance by enabling Active Directory replication, ensuring that directory data is synchronized across both servers. If one domain controller fails, the other can continue to handle authentication and directory services without interruption. This approach significantly improves the availability, resilience, and reliability of the overall infrastructure.

Before setting up DC02 as an additional domain controller, ensure the following prerequisites are completed:
* Rename the server to DC02
* Assign a static IP address of 192.168.0.3
* Join the server to the domain (int.homelab.local)

Once these steps are complete, install the Active Directory Domain Services (AD DS) role and select Promote this server to a domain controller. After the installation finishes, launch the Deployment Configuration Wizard to begin configuring DC02 as an additional domain controller.
![DC02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/2501c2f2cbcaf196e8334927a6b774897eec3632/images/DC02%20(1).PNG)

Since the server is already joined to int.homelab.local, the domain is automatically detected and ready to use; this is why joining the domain beforehand simplifies the process.
![DC02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/8c8859cd199459ff480a00fd9dd10457818c4fe9/images/DC02%20(2).PNG)

Keep DNS Server selected (DC02 will also act as a DNS server). Keep Global Catalog (GC) enabled so it holds a full copy of directory objects. Leave Read-Only Domain Controller (RODC) unchecked, as we want this to be a fully writable domain controller. Set a password > Next.
![DC02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/8c8859cd199459ff480a00fd9dd10457818c4fe9/images/DC02%20(3).PNG)

We can ignore this warning about DNS delegation since DNS is being handled internally in this lab.
![DC02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/8c8859cd199459ff480a00fd9dd10457818c4fe9/images/DC02%20(4).PNG)

Next, choose a replication source. The wizard will automatically detect DC01 as the existing domain controller, and DC02 will replicate all Active Directory data from it. Select DC01 and Next.
![DC02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/8c8859cd199459ff480a00fd9dd10457818c4fe9/images/DC02%20(5).PNG)

Leave the default database, log, and SYSVOL paths.
![DC02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/8c8859cd199459ff480a00fd9dd10457818c4fe9/images/DC02%20(6).PNG)

Next.
![DC02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/8c8859cd199459ff480a00fd9dd10457818c4fe9/images/DC02%20(7).PNG)

After completing the prerequisite check, confirm that all requirements have passed (warnings are acceptable). Then proceed with the installation. Once complete, the server will reboot automatically. After reboot, DC02 will be fully configured as an additional domain controller.
![DC02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/8c8859cd199459ff480a00fd9dd10457818c4fe9/images/DC02%20(8).PNG)

To verify replication, I created a new Organizational Unit (OU) named testdc02 and added a user called testuser from DC02.
![DC02test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/35c54da17cfb4d502756be1a0d8cc73d07ea6a36/images/testDC02%20(1).PNG)

![DC02test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/35c54da17cfb4d502756be1a0d8cc73d07ea6a36/images/testDC02%20(2).PNG)

When I checked DC01, the same OU and user were visible there as well. This confirms that Active Directory replication is functioning correctly, and both domain controllers are synchronizing changes. In other words, both DCs are communicating with each other and maintaining a consistent copy of the directory, which is exactly what we want for redundancy and high availability.
![DC02test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/35c54da17cfb4d502756be1a0d8cc73d07ea6a36/images/testDC02%20(3).PNG)

When checking DNS on DC02, you can see that all the records and zones are already present—even though we didn’t manually configure DNS on this server.
![DC02testDNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/29dd00d0f3c99b9597bc52fe7ac9c50ed614465f/images/DNStestDC02%20(1).PNG)

![DC02testDNS](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/29dd00d0f3c99b9597bc52fe7ac9c50ed614465f/images/DNStestDC02%20(2).PNG)

**DHCP02 (Secondary / Backup DHCP Server)**
Before setting up DHCP02 as a Backup DHCP Server, ensure the following prerequisites are completed:
* Rename the server to DHCP02
* Assign a static IP address of 192.168.0.4
* Join the server to the domain (int.homelab.local)

To install DHCP: From Server Manager's menu bar, click on Manage > "Add Roles and Features" Next > select a role-based installation, verify that the correct server (DHCP02 – 192.168.0.4) was selected, and then choose the DHCP Server role along with its management tools. After installation, complete the post-deployment configuration, which creates the necessary security groups and authorizes the DHCP server in Active Directory, the same as before. 
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(1).PNG)

At this point, the DHCP role is installed and active, but there are no scopes configured, meaning the server is not yet distributing IP addresses. In a typical setup, a new scope would be created. However, in this case, DHCP02 is not intended to act as a standalone DHCP server. Instead, it will function as a failover partner to the primary DHCP server (running on DC01). The scopes will be shared through a DHCP failover configuration, allowing both servers to provide IP address leasing and ensure high availability if one server goes offline.

To configure DHCP redundancy, we set up a failover relationship between DC01 (DHCP1) and DHCP02. From DC01, navigate to your existing DHCP scope (192.168.0.0/24), right-click it, and select Configure Failover. 
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(2).PNG)

This launches the failover wizard.
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(3).PNG)

Next, specify the partner server. You can either enter the IP address (192.168.0.4), or...
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(4).PNG)

...browse and select DHCP02 from the environment. Click on Add server > browse and type DHCP02 and check names. Then hit OK. 
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(5).PNG)

![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(6).PNG)

Relationship Name: You can keep the default or rename it for clarity. 
Mode Selection:
* Load Balance → Both servers actively respond to DHCP requests
* Hot Standby → One server is active, the other waits for failover

In this lab, Load Balance is selected, allowing both servers to distribute IP addresses simultaneously, improving performance and availability.
Load Distribution: Default 50/50 split (both servers share the workload equally). Configure a shared secret for authentication between the two servers.
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(7).PNG)

After reviewing the summary, click Finish.
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(8).PNG)

The failover relationship is created almost instantly.
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(9).PNG)

On DC01, you can confirm the configuration by checking the scope properties
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(10).PNG)

Failover tab, where the partner server is listed: dhcp02.int.homelab.local
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(11).PNG)

Switch to DHCP02. Initially, there were no scopes configured. Click on Refresh.
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(12).png)

After refreshing, the scope is automatically replicated from DC01. You will see the same configuration, including the IP range: 192.168.0.30 – 192.168.0.230
![DHCP02](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/c14c539e577883270189d5ca5698904d09bdcc80/images/DHCP%202%20(13).PNG)


**To validate the DHCP failover setup**, I performed a failover test by simulating a failure of the primary server (DC01). Initially, when checking PC001 with ipconfig, the DHCP server was 192.168.0.2 (DC01), as expected. To test redundancy, I disabled the network adapter on DC01, effectively taking the primary DC and DHCP server offline. This forces the failover partner (DHCP02) to take over DHCP responsibilities.
![DHCP02test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/d08f0b0533fffd24b1d24ea7e5629d6982afa75d/images/test%20(1).PNG)


Next, on PC001, I ran ```ipconfig /release``` to remove the current IP configuration and ran ```ipconfig``` again to see the self-assigned IP.
![DHCP02test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/d08f0b0533fffd24b1d24ea7e5629d6982afa75d/images/test%20(2).PNG)


Then ```ipconfig /renew``` to request a new IP address from a DHCP server
![DHCP02test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/d08f0b0533fffd24b1d24ea7e5629d6982afa75d/images/test%20(3).PNG)


After renewing, the client successfully received a new IP address. To verify which server responded, I ran ```ipconfig /all``` The result showed that the DHCP server was 192.168.0.4, which is DHCP02.
![DHCP02test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/d08f0b0533fffd24b1d24ea7e5629d6982afa75d/images/test%20(4).PNG)

This confirms that the failover configuration is working correctly. DHCP02 detected that DC01 was offline and DHCP02 successfully took over and continued assigning IP addresses. Even with the primary server unavailable, the network remained fully functional. This demonstrates true high availability and resilience in the DHCP infrastructure.

Lastly, we will test whether a new user can log in to the domain for the first time while DC01 is offline, with only DC02 and DHCP02 remaining operational.

Before performing this test, we need to ensure proper DNS redundancy is in place. On the DHCP server, we will configure the DNS server options so that clients are not solely dependent on DC01 for name resolution. This ensures that when DC01 is unavailable, clients still know where to query for DNS services.

On DC01, open Server Manager → Tools → DHCP. In the DHCP console, expand the server, then expand IPv4, and then the configured Scope. Click on Scope Options. Based on the current configuration, we can see that the primary DNS server listed is DC01 (192.168.0.2).
![DHCP-DNS-Config](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/b1f0bd497e8db3a6ac11e2940d13024201113319/images/DHCP%20DNS%20config%20(1).PNG)

Double-click Option 006 – DNS Servers.
![DHCP-DNS-Config](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/b1f0bd497e8db3a6ac11e2940d13024201113319/images/DHCP%20DNS%20config%20(2).PNG)

To add DC02 as a secondary DNS server, either:
Enter the IP address 192.168.0.3 and click Add, or
Enter the hostname DC02, click Resolve, and then Add
![DHCP-DNS-Config](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/b1f0bd497e8db3a6ac11e2940d13024201113319/images/DHCP%20DNS%20config%20(3).PNG)

Once added, click Apply to save the changes.
![DHCP-DNS-Config](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/b1f0bd497e8db3a6ac11e2940d13024201113319/images/DHCP%20DNS%20config%20(4).PNG)

After completing this on DC01, repeat the same configuration on DHCP02. This guarantees that clients will always receive both DNS servers (DC01 and DC02) regardless of which DHCP server responds to their request.
![DHCP-DNS-Config](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/b1f0bd497e8db3a6ac11e2940d13024201113319/images/DHCP%20DNS%20config%20(5).PNG)

I created a test user named “Sam” on DC02.
![OfflineDC01test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/223ec35dc6d0d54fc0bc1086ba0c4578249e7355/images/testing%20offline%20DC01%20(1).PNG)

After refreshing Active Directory Users and Computers on DC01, I can now also see the Sam account on DC01. As expected, this confirms that both domain controllers are fully synchronized through Active Directory replication, and they function as redundant copies of each other within the environment.
![OfflineDC01test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/223ec35dc6d0d54fc0bc1086ba0c4578249e7355/images/testing%20offline%20DC01%20(2).PNG)

I powered off DC01, leaving only DC02 and DHCP02 running, and attempted to log in to the domain for the first time using the Sam user account on the Windows 10 Pro client machine.
![OfflineDC01test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/223ec35dc6d0d54fc0bc1086ba0c4578249e7355/images/testing%20offline%20DC01%20(3).PNG)

The login was successful, confirming that authentication is fully handled by DC02 in the absence of DC01.
![OfflineDC01test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/223ec35dc6d0d54fc0bc1086ba0c4578249e7355/images/testing%20offline%20DC01%20(4).PNG)


Running ```ipconfig /all``` on the client shows the updated network configuration: DHCP Server: 192.168.0.4 (DHCP02); DNS Servers: 192.168.0.2 and 192.168.0.3
![OfflineDC01test](https://github.com/AyboFrankOz/HyperV-ADDS-DNS-DHCP-Redundancy-Lab/blob/223ec35dc6d0d54fc0bc1086ba0c4578249e7355/images/testing%20offline%20DC01%20(5).PNG)

This confirms that: DHCP failover is working correctly, with DHCP02 serving IP addresses when DC01 is offline. DNS redundancy is in place, allowing clients to resolve domain resources using either domain controller. Active Directory authentication continues to function normally through DC02. Overall, the environment successfully maintains full functionality even with the primary domain controller offline, demonstrating a properly configured, redundant, and highly available infrastructure.

## Conclusion ##
This lab demonstrates the end-to-end deployment of a resilient Windows Server infrastructure using Hyper-V. Starting from a fully isolated virtual network, I configured core services including Active Directory, DNS, and DHCP, and then enhanced the environment by implementing redundancy for both domain controllers and DHCP services.

By adding a secondary domain controller (DC02), I ensured that Active Directory and DNS services remain available through replication and fault tolerance. Similarly, configuring DHCP failover with DHCP02 provided continuous IP address allocation even when the primary server (DC01) was taken offline.

Testing the environment by simulating failures confirmed that both Active Directory replication and DHCP failover function as expected, maintaining service availability without disruption. Additionally, a final validation test was performed by powering off DC01 and logging into the domain with a newly created user from a client machine. The successful login, along with DHCP and DNS services being handled by DHCP02 and DC02, further confirmed that the infrastructure continues to operate seamlessly even when the primary domain controller is unavailable.

Overall, this lab highlights the importance of redundancy, high availability, and proper network design in real-world IT environments, while providing hands-on experience with enterprise-level infrastructure concepts.
