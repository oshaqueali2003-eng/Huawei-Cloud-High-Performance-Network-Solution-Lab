# Huawei-Cloud-High-Performance-Network-Solution-Lab
Huawei Cloud High-Performance Network Solution Lab  A hands-on lab where I designed a small but realistic cloud network on Huawei Cloud (region: AP-Singapore). A Discuz! forum website acts as the "service system". Around it I built the database, internet access, a private O&amp;M (operations) network, and private DNS.
Step-by-Step Guide

Region for everything: AP-Singapore. Replace placeholders like <ecs-web-private-ip> with your own values. Never commit real passwords.

Part 1: Basic environment
Step 1. Create subnet-db and sg-db

Why? The database should live in its own subnet with its own firewall, separate from the web server.

Console, then Virtual Private Cloud. You will see vpc-primary.
Subnets, then Create Subnet:
VPC: vpc-primary
Name: subnet-db
IPv4 CIDR Block: 192.168.2.0/24
Access Control, then Security Groups, then Create Security Group:
Name: sg-db
Template: Fast-add rule, inbound MySQL (3306)
Open sg-db, then Inbound Rules. On the TCP 3306 rule click Modify. Change Source from an IP address to Security group and choose sg-web. Confirm.

Result: only the web server group can reach the database port.

Step 2. Buy the RDS database
Relational Database Service, then Buy DB Instance.
Settings:
Billing: Pay-per-use
Region: AP-Singapore
Engine: MySQL 5.7
Instance type: Primary/Standby (primary and standby in different AZs)
Storage: Cloud SSD, 40 GB
Class: Dedicated, 2 vCPUs / 4 GB
VPC / subnet: vpc-primary / subnet-db
Port: 3306
Security group: sg-db
Password: choose a strong one and store it safely
Submit and wait for status Available. Note the floating IP of the database.
Step 3. Install Discuz! and connect to RDS
ECS console, copy the EIP of ecs-web.
Browser: http://<EIP>:80.
Accept the license. All directory checks must show Can write. Click Next.
Choose A new installation Discuz! X (Contain UCenter Server).
Fill in:
Database server: the RDS floating IP
Database password: the RDS root password
Admin password: a password for the forum admin
Finish. Open http://<EIP>:80 again and log in as admin.
Part 2: Internet access with EIP and NAT Gateway
Step 1. Unbind the EIP

ECS console, then ecs-web, then More, Manage Network, Unbind EIP. Reload the site. It should now fail, which proves the server has no internet path.

Step 2. Create the NAT Gateway

NAT Gateway, then Buy Public NAT Gateway:

Name: nat-web
VPC: vpc-primary, Subnet: subnet-web
Billing: Pay-per-use
Step 3. Add the SNAT rule (outbound)
Scenario: VPC
CIDR: existing, subnet-web (192.168.1.0/24)
Public IP: the EIP you just unbound
Step 4. Add the DNAT rule (inbound)
Scenario: VPC / Server, instance ecs-web
Public IP: the same EIP
Outside port: 80, Inside port: 80, Protocol: TCP
Step 5. Test

Reload http://<EIP>. The Discuz! home page should load again, now through the NAT Gateway.

Part 3: VPC peering between vpc-primary and vpc-om
Step 1. Create the ops network
Create VPC: name vpc-om, CIDR 172.16.0.0/16, default subnet subnet-om with 172.16.1.0/24.
Create security group sg-om with the General-purpose web server template.
Step 2. Create the jump server

Buy ECS:

Spec: s6.large.2
Image: CentOS 7.6 64-bit
Network: vpc-om / subnet-om
Security group: sg-om
EIP: not required (we log in through the console)
Name: ecs-om
Login: password
Step 3. Create the peering connection

VPC Peering Connections, then Create:

Local VPC: vpc-primary
Account: My account
Peer project: ap-southeast-3
Peer VPC: vpc-om

Click Add Now to add routes:

Route table	Destination	Next hop
vpc-primary	vpc-om CIDR (172.16.0.0/16)	the peering connection
vpc-om	vpc-primary CIDR (192.168.0.0/16)	the peering connection
Step 4. Verify connectivity
Open sg-web, then Inbound Rules, Add Rule: Protocol ALL, Source Security group sg-om.
Note the private IP of ecs-web.
Remote Login to ecs-om (Other Login Modes, user root).
Run:
bash
ping -c 3 <ecs-web-private-ip>

Expected: 3 replies, 0% packet loss.

Part 4: DNS design

Why? Admins should not memorize IPs. A name is easier.

Step 1. Create the private zone
Domain Name Service, then Private Zones, then Create Private Zone.
Domain name: www.abc.com, VPC: vpc-om.
Manage Record Sets, then Add Record Set:
Type: A
TTL: default
Value: private IP of ecs-web
Name: leave blank (so the record is for www.abc.com itself)
Step 2. Verify from ecs-om
bash
ping -c 3 www.abc.com
