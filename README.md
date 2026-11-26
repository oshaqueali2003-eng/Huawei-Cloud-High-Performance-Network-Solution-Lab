# Huawei-Cloud-High-Performance-Network-Solution-Lab
Huawei Cloud High-Performance Network Solution Lab  A hands-on lab where I designed a small but realistic cloud network on Huawei Cloud (region: AP-Singapore). A Discuz! forum website acts as the "service system". Around it I built the database, internet access, a private O&amp;M (operations) network, and private DNS.

# Huawei Cloud: High-Performance Network Solution Lab

## 📌 Overview
This repository contains the complete implementation of a High-Performance Network Solution lab on Huawei Cloud, deployed in the AP-Singapore region. The main objective of this project was to design a secure, scalable, and highly available network architecture from scratch. The environment hosts a Discuz! Forum web application, which is backed by a highly available RDS MySQL database. To ensure strict security and separation of duties, I built an isolated Operations and Maintenance (O&M) network and connected it to the main network using VPC Peering. Secure internet access is managed efficiently through a Public NAT Gateway, and Private DNS is configured for seamless internal name resolution.

## 🏗️ Architecture Diagram
![Architecture Diagram](architecture.png)
*(Note: The image file should be named `architecture.png` in the root of this repository)*

## 🌐 Network Design and CIDR Plan
The network is logically divided into two separate Virtual Private Clouds (VPCs) to ensure isolation. The primary VPC (`vpc-primary`) uses the CIDR block `192.168.0.0/16` and hosts the main web and database resources. Within this VPC, the web server is placed in `subnet-web` (`192.168.1.0/24`), while the database resides in a completely isolated `subnet-db` (`192.168.2.0/24`). For management purposes, a separate O&M VPC (`vpc-om`) was created using the `172.16.0.0/16` CIDR block, containing `subnet-om` (`172.16.1.0/24`) which hosts the Jump Server. 

## 🚦 Key Configurations and Implementations

**Database Isolation and Security:** To protect sensitive data, the database is not exposed to the internet. I deployed a Primary/Standby RDS MySQL 5.7 instance across different Availability Zones for high availability. A dedicated Security Group (`sg-db`) was configured to allow MySQL traffic (Port 3306) strictly from the web server's Security Group (`sg-web`), ensuring no unauthorized access.

**Secure Internet Access via NAT Gateway:** Initially, the EIP was unbound from the web server to prove internet isolation. To provide secure outbound internet access for updates, I deployed a Public NAT Gateway. I configured a SNAT rule to allow instances in the web subnet to access the internet. Additionally, a DNAT rule was created to map the NAT Gateway's public IP (Port 80) to the private IP of the web server (Port 80), allowing external users to access the Discuz! forum without directly exposing the server.

**VPC Peering and Cross-Network Routing:** To allow the O&M team to manage the primary network, I established a VPC Peering connection between `vpc-primary` and `vpc-om`. I carefully updated the Route Tables in both VPCs to ensure bidirectional traffic flow. Furthermore, I modified the `sg-web` inbound rules to allow ICMP and SSH traffic originating only from the O&M security group (`sg-om`).

**Private DNS Design:** To simplify internal management, I configured a Private DNS Zone for `abc.com` within `vpc-om`. I created an A-record mapping `www.abc.com` to the private IP of the web server (`192.168.1.10`). This allows administrators to securely access services using easy-to-remember domain names instead of complex IP addresses.

## 🧪 Connectivity Tests (Evidence)
To validate the architecture, several connectivity tests were performed. From the Jump Server (`ecs-om`), I successfully pinged the private IP of the web server, confirming that VPC Peering and routing were working perfectly. I also pinged `www.abc.com` from the Jump Server, which successfully resolved to the correct private IP, proving the Private DNS configuration was accurate. Finally, the Discuz! forum was accessed via a web browser using the NAT Gateway's Public IP, which confirmed that the DNAT rules and database connections were functioning flawlessly.

## 🧹 Cleanup Steps
After completing the lab and capturing the necessary evidence, all resources were properly cleaned up to avoid unnecessary billing. This included deleting the Discuz! application, releasing the EIP, deleting the Public NAT Gateway, removing the VPC Peering connection, and deleting the Private DNS Zone. Finally, the ECS instances and RDS database were terminated, followed by the deletion of Security Groups and VPCs.







