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

## Domain Name System (DNS)

When installing Active Directory Domain Services (AD DS), the DNS Server role is installed and configured automatically. DNS plays a critical role in Active Directory by enabling clients to locate domain controllers and other services within the domain. Without a properly functioning DNS infrastructure, core Active Directory operations such as authentication, replication, and service discovery would fail. Let’s take a quick look at what Active Directory automatically configures in DNS during deployment:
