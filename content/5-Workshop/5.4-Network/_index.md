---
title : "Networking"
date : 2024-01-01
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

## 1. Overview

The main objective of this section is to create an isolated network environment that supports Internet connectivity while keeping the system's services secure.
In this section, I configured one VPC, four subnets, one Internet Gateway, two NAT Gateways, three Route Tables, and two Security Groups.
These resources provide the networking foundation for deploying subsequent components such as EKS, the RDS database, and the caching system.
All resources are deployed in the Singapore Region, across two Availability Zones: `ap-southeast-1a` and `ap-southeast-1b`.

## 2. Create a VPC – The System's Private Network

### 2.1. Introduction to VPC

First, I set up a private network for the RideNow system using the AWS VPC service.

In RideNow, the VPC serves as the networking foundation for deploying subsequent components such as EKS, the RDS database, and the caching system.

### 2.2. Steps to Create a VPC

To begin, I sign in to the AWS Management Console and select the **Asia Pacific (Singapore)** Region, whose code is `ap-southeast-1`.
Then, I search for the VPC service, navigate to **Your VPCs**, and choose **Create VPC** to create the private network.

![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-vpc.png)
Here, the CIDR block `10.30.0.0/16` defines the VPC's IP address range. This address range can then be divided into smaller networks, called subnets, for different system components.

### 2.3. Configure DNS for the VPC

After successfully creating the VPC, I configure DNS to support domain name resolution within the network.
First, I navigate to **Your VPCs**, select `ridenow-prod-vpc`, and choose **Actions → Edit VPC settings**.
Under DNS Settings, I enable two features:
**The first is DNS Resolution.**
This feature allows resources within the VPC to use AWS-provided DNS resolution to translate domain names into IP addresses when needed.
**The second is DNS Hostnames.**
This feature supports the assignment of DNS hostnames to eligible resources within the VPC.
Enabling these two DNS features makes it easier for system components to communicate using domain names instead of directly using IP addresses.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-enable-vpc-dns-settings.png)

### 2.4. Verify the VPC Using AWS CLI on Ubuntu WSL

In addition to checking the configuration directly in the AWS Console, I use the AWS CLI in Ubuntu WSL to verify VPC information from the command line.
This lets me confirm resource configuration directly through the AWS API rather than relying solely on the management interface.
**First, I retrieve the VPC ID using its configured name.**
I use the following command:
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-find-vpc-id.png)

Here, `describe-vpcs` queries VPC information, while `--filters` identifies the correct VPC using its Name tag.
Next, I run:
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-display-vpc-id.png)
This command displays the VPC ID automatically assigned by AWS.

**Next, I verify the VPC's CIDR block and state.**
I run the following command:
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-verify-vpc-cidr-and-state.png)
The result shows that the VPC has the intended CIDR block and is in the Available state.

**Finally, I verify the two DNS features.**
To check DNS Resolution, I use:
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-verify-vpc-dns-resolution.png)

Next, to check DNS Hostnames, I use:
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-verify-vpc-dns-hostnames.png)

For both commands, I verify that the returned `Value` is `true`.
This confirms that both DNS features are enabled on the VPC.

### 2.5. Results and Next Steps

After completing the deployment and verification, I have created a VPC named `ridenow-prod-vpc`, with the CIDR block `10.30.0.0/16`, in the Singapore Region.
I have also enabled DNS Resolution and DNS Hostnames and used the AWS CLI to verify the VPC's state and attributes.
This VPC provides the networking foundation for the entire RideNow system.
However, at this point, I have only created the private network and have not yet established an Internet connection for the VPC.

Therefore, **the next step is to create an Internet Gateway and attach it to the VPC**, preparing for Internet connectivity for the system's public subnets.

## 3. Create an Internet Gateway – Connect the VPC to the Internet

### 3.1. Introduction to Internet Gateway

After creating the VPC, I configure an Internet Gateway for the RideNow system.
An Internet Gateway acts as a connection point between the VPC network and the Internet.

In RideNow, the Internet Gateway supports Internet access for resources in public subnets when those resources have appropriate IP addresses, routes, and access permissions.

### 3.2. Steps to Create an Internet Gateway

First, I sign in to the AWS Management Console and select the **Asia Pacific (Singapore)** Region, `ap-southeast-1`.

On the Internet Gateway creation page, I enter the resource name `ridenow-prod-igw`.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-internet-gateway.png)

Here, `ridenow` is the system name, `prod` represents the production environment, and `igw` stands for Internet Gateway.
After reviewing the information, I select **Create internet gateway** to create the resource.

### 3.3. Attach the Internet Gateway to the VPC

After the Internet Gateway is created, its initial state is **Detached**, meaning it is not associated with any VPC.
I therefore attach it to the VPC created in the previous step.
First, I select `ridenow-prod-igw` and choose **Actions → Attach to VPC**.
Under Available VPCs, I select `ridenow-prod-vpc` and click **Attach internet gateway**.
Once the operation succeeds, the Internet Gateway's state changes from **Detached** to **Attached**.
This confirms that the Internet Gateway is attached to the correct RideNow VPC.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-attach-internet-gateway-to-vpc.png)

### 3.4. Results and Next Steps

After deployment and verification, I have created the Internet Gateway `ridenow-prod-igw` in the Singapore Region.
It has been attached to the VPC `ridenow-prod-vpc`.
I also used the AWS CLI to check and confirm the attachment state of these two resources.
However, **creating and attaching an Internet Gateway does not immediately mean that resources inside the VPC can access the Internet**.
This also requires suitable subnets, Route Tables that direct traffic to the Internet Gateway, and other necessary network settings.

Therefore, the next step is to create public and private subnets to segment the network, then configure Route Tables to establish Internet routing for the RideNow public subnets.

## 4. Create Subnets – Network Segmentation

### 4.1. Introduction to Subnets

After creating the VPC and configuring the Internet Gateway, I divide the network by creating subnets.
A subnet is a smaller network within a VPC. It allows us to divide an IP address range into separate network segments for different purposes.

In RideNow, I divide `ridenow-prod-vpc` into **four subnets**: two public subnets and two private subnets.
Public subnets are intended for resources that need a direct route to the Internet, such as NAT Gateways and public-facing Load Balancers.
Private subnets are used for internal components such as applications and databases, reducing direct exposure to the Internet.

This segmentation creates a clearer network structure and makes resources easier to manage and secure.

### 4.2. Steps to Create Subnets in the AWS Console

First, I sign in to the AWS Management Console and select the **Asia Pacific (Singapore)** Region, `ap-southeast-1`.

Under VPC ID, I select the previously created `ridenow-prod-vpc`.
Next, I configure the four subnets as follows:
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-1.png)

![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-2.png)

![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-3.png)

![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-4.png)

![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-5.png)

![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-6.png)

| Subnet | Availability Zone | IPv4 CIDR |
| --- | --- | --- |
| `ridenow-prod-public-1` | ap-southeast-1a | `10.30.0.0/20` |
| `ridenow-prod-public-2` | ap-southeast-1b | `10.30.16.0/20` |
| `ridenow-prod-private-1` | ap-southeast-1a | `10.30.48.0/20` |
| `ridenow-prod-private-2` | ap-southeast-1b | `10.30.64.0/20` |

The two public subnets are placed in different Availability Zones, as are the two private subnets.
Availability Zones are independent infrastructure locations within the same AWS Region.
Distributing subnets across two Availability Zones lays the groundwork for a highly available architecture. If one Availability Zone experiences an outage, components properly deployed in the other Availability Zone may continue serving the system.
After entering each subnet's name, Availability Zone, and IPv4 CIDR, I choose **Add new subnet** to add the remaining subnets.
Finally, I click **Create subnet** to create all four subnets.

### 4.3. Configure Kubernetes Tags for the Subnets

After creating the subnets, I configure Kubernetes tags.
These tags help the AWS Load Balancer Controller identify appropriate subnets when deploying Load Balancers on EKS in later steps.
First, I navigate to **VPC → Subnets**, select each subnet, and open the **Tags** tab to inspect or add the required tags.
For the two public subnets, I use the following tag:
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-public-subnet-kubernetes-tags.png)

This tag identifies subnets suitable for deploying public, or Internet-facing, Load Balancers.

For the two private subnets, I use the following tag:

![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-private-subnet-kubernetes-tags.png)
This tag identifies subnets suitable for deploying internal Load Balancers.

### 4.4. Configure Automatic Public IP Assignment

Next, I configure automatic public IPv4 address assignment for the subnets.
For the two public subnets, I enable **Enable auto-assign public IPv4 address**.
To do this, I navigate to **VPC → Subnets**, select `ridenow-prod-public-1`, and choose **Actions → Edit subnet settings**.
I select **Enable auto-assign public IPv4 address** and click **Save**.
I repeat the procedure for `ridenow-prod-public-2`.
This setting allows EC2 instances launched in the public subnets to automatically receive public IPv4 addresses by default, unless another configuration overrides it.

In contrast, I keep automatic public IPv4 assignment disabled for `ridenow-prod-private-1` and `ridenow-prod-private-2`.
This reduces the chance of internal resources unintentionally receiving public IP addresses.
However, enabling public IP assignment alone does not provide immediate Internet connectivity. Appropriate Route Tables and other networking conditions are still required.

### 4.5. Results and Next Steps

After deployment and verification, I have created four subnets in `ridenow-prod-vpc`: two public subnets and two private subnets.
These subnets are distributed across `ap-southeast-1a` and `ap-southeast-1b`.
I also configured Kubernetes tags to support future Load Balancer deployments on EKS and enabled automatic public IP assignment for the two public subnets.
Afterward, I used the AWS Console and AWS CLI to verify each subnet's name, CIDR block, Availability Zone, state, public IP settings, and tags.
This network segmentation clearly separates public and internal components and establishes a foundation for running services across multiple Availability Zones.
However, at this stage, the subnets have only been created and given their basic settings; their intended Internet routes are not yet fully configured.
Therefore, **the next step is to create a Public Route Table**, add the default route `0.0.0.0/0` pointing to `ridenow-prod-igw`, and associate the Route Table with both public subnets to complete their Internet routing.

## 5. Create Route Tables – Direct Network Traffic

### 5.1. Introduction to Route Tables

After creating the four subnets, I configure Route Tables for the RideNow system.
A Route Table in AWS VPC determines where packets are routed when they are sent from subnets to other network destinations.

For RideNow, I designed three Route Tables: one Public Route Table and two Private Route Tables.
The Public Route Table directs Internet-bound traffic from both public subnets through the Internet Gateway.
The two Private Route Tables are configured separately for each private subnet, using NAT Gateways so internal resources can initiate outbound Internet connections without public IP addresses.

At this step, I create and configure the Public Route Table first. The two Private Route Tables will be completed after the NAT Gateways are deployed.

### 5.2. Steps to Create a Public Route Table in the AWS Console

First, I sign in to the AWS Management Console and select the **Asia Pacific (Singapore)** Region, `ap-southeast-1`.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-public-route-table.png)
On the Route Table creation page, I enter the following settings:
- **Name:** `ridenow-prod-public-rt`.
- **VPC:** Select `ridenow-prod-vpc`.

After creating the Route Table, I open the **Routes** tab to inspect its existing routes.
At this point, the Route Table already contains a default local route for the VPC:
```
Destination: 10.30.0.0/16
Target: local
```
Here, `10.30.0.0/16` is the RideNow VPC address range, while `local` allows resources within the VPC to route traffic to each other.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-public-route-table-local-route.png)
AWS automatically creates this local route when the Route Table is created.

### 5.3. Configure the Internet Route

After creating the Public Route Table, I configure a route that allows public subnets to use the Internet Gateway.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-public-route-table-internet-gateway-route.png)
Here, `0.0.0.0/0` represents the default route covering all IPv4 addresses.
This means that when a packet's destination does not match a more specific route, the Route Table sends the traffic to the Internet Gateway.
The Internet Gateway `ridenow-prod-igw`, created and attached to the VPC earlier, provides the connection between the VPC and the Internet.
After saving, the Public Route Table has two main routes:

| Destination | Target |
| --- | --- |
| `10.30.0.0/16` | `local` |
| `0.0.0.0/0` | `ridenow-prod-igw` |

The `local` route supports communication inside the VPC, while `0.0.0.0/0` routes Internet-bound traffic through the Internet Gateway.

### 5.4. Associate the Public Route Table with Both Public Subnets

After setting up Internet routing, I associate the Public Route Table with the two previously created public subnets.

Under Explicit subnet associations, I associate both public subnets with the same Public Route Table.
- `ridenow-prod-public-1`.
- `ridenow-prod-public-2`.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-public-route-table-subnet-associations.png)

When resources in either public subnet need to send traffic to the Internet, the Route Table provides a default route to the Internet Gateway.
However, actual Internet access still requires an appropriate public IP address and Security Group and Network ACL settings that permit the traffic.

### 5.5. Results and Next Steps

After deployment and verification, I have created a Public Route Table named `ridenow-prod-public-rt` in `ridenow-prod-vpc`.
I configured its default route `0.0.0.0/0` to point to the Internet Gateway `ridenow-prod-igw`.
I also associated this Route Table with the two public subnets, `ridenow-prod-public-1` and `ridenow-prod-public-2`.
Finally, I used the AWS Console and AWS CLI on Ubuntu WSL to verify the routes and the Route Table's subnet associations.
The Public Route Table establishes Internet routing for the two public subnets and clearly separates the public network area from the internal one.
However, **only the Public Route Table is complete at this stage; the two Private Route Tables have not yet been fully configured**.
For resources in the private subnets to initiate outbound Internet access without using public IP addresses, NAT Gateways are required.
Therefore, the next step is to **allocate two Elastic IP addresses and create two NAT Gateways**, with one NAT Gateway in a public subnet in each Availability Zone.

After creating the NAT Gateways, I will create the two Private Route Tables and configure Internet routing through their corresponding NAT Gateways.

## 6. Create NAT Gateways – Enable Internet Access for Private Subnets

### 6.1. Introduction to NAT Gateway

After configuring the Public Route Table, I deploy NAT Gateways for the RideNow system.

A NAT Gateway is an AWS networking service that allows resources in private subnets to initiate outbound Internet connections without having directly assigned public IP addresses.
In RideNow, internal components such as applications running on EKS may need Internet access to download libraries, retrieve container images, or connect to external services.

However, for security reasons, these components are designed to run in private subnets without public IP addresses.
Therefore, I use NAT Gateways to provide outbound Internet connectivity to these resources.
Specifically, I allocate two Elastic IP addresses, create two NAT Gateways, and configure two Private Route Tables.
Each NAT Gateway is deployed in a public subnet in a different Availability Zone to reduce dependence on a single NAT Gateway.

### 6.2. Allocate Two Elastic IP Addresses

First, I allocate two Elastic IP addresses for the two NAT Gateways.
An Elastic IP is a static public IPv4 address allocated by AWS. It gives a NAT Gateway a stable public IP address for outbound Internet connectivity.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-allocate-nat-elastic-ip-1.png)

After entering the required information, I click **Allocate** to allocate the first Elastic IP address.

Next, I repeat the procedure for the second Elastic IP, changing the Name tag to `ridenow-prod-nat-eip-2`.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-allocate-nat-elastic-ip-2.png)

Afterward, I return to the Elastic IPs list to verify the resources.
I confirm that two Elastic IP addresses have been allocated and that each has a unique Allocation ID generated by AWS.
These two Elastic IPs will be associated with the first and second NAT Gateways, respectively.

### 6.3. Create the First NAT Gateway

After allocating the two Elastic IPs, I create the first NAT Gateway.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-nat-gateway-1.png)

The first NAT Gateway is placed in Public Subnet 1, in Availability Zone `ap-southeast-1a`.
I select Public as its Connectivity type because it needs to reach the Internet through the VPC's Internet Gateway.
I also assign the first Elastic IP to give the NAT Gateway a public IPv4 address.

### 6.4. Create the Second NAT Gateway

Next, I create the second NAT Gateway to serve the private subnet in the other Availability Zone.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-nat-gateway-2.png)

Unlike the first NAT Gateway, the second is placed in Public Subnet 2, in Availability Zone `ap-southeast-1b`.
Each NAT Gateway uses its own Elastic IP and is deployed in a separate Availability Zone.

This design reduces the system's dependence on a single NAT Gateway and allows each private subnet to use a NAT Gateway in the same Availability Zone.

### 6.5. Create the First Private Route Table

Once both NAT Gateways are ready, I configure Route Tables so that the private subnets can reach the Internet through their NAT Gateways.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-private-route-table-1.png)

I configure the following route:
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-private-route-table-1-nat-default-route.png)
Here, `0.0.0.0/0` is the default route for IPv4 destinations that do not match a more specific route.
The NAT Gateway `ridenow-prod-nat-1` receives Internet-bound traffic from Private Subnet 1.

Finally, I associate this Route Table with `ridenow-prod-private-1`.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-private-route-table-1-subnet-association.png)

Private Subnet 1 now has a default route through NAT Gateway 1.

### 6.6. Create the Second Private Route Table

Next, I create the second Private Route Table for Private Subnet 2.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-private-route-table-2.png)

I then associate it with `ridenow-prod-private-2`.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-private-route-table-2-subnet-association.png)

Both private subnets are now configured to use separate NAT Gateways.
Specifically, Private Subnet 1 uses NAT Gateway 1 in Availability Zone 1a, while Private Subnet 2 uses NAT Gateway 2 in Availability Zone 1b.

### 6.7. Results and Next Steps

After deployment and verification, I have allocated two Elastic IP addresses and created two NAT Gateways for RideNow.
The first NAT Gateway is in Public Subnet 1 in Availability Zone `ap-southeast-1a`, and the second is in Public Subnet 2 in Availability Zone `ap-southeast-1b`.
Each NAT Gateway uses its own Elastic IP for Internet connectivity.
I have also created two Private Route Tables and configured their default routes `0.0.0.0/0` to point to the corresponding NAT Gateways.
With this configuration, resources in the private subnets have outbound routes to the Internet without requiring directly assigned public IPs, provided other networking and security requirements are met.
Deploying NAT Gateways across two Availability Zones also reduces dependence on one NAT Gateway and establishes a foundation for high availability.
Finally, I used the AWS Console and AWS CLI on Ubuntu WSL to check the NAT Gateways' states and Route Table configurations.
**After completing the NAT Gateway setup, the next step is to configure Security Groups** to control network traffic between the application and database components of RideNow.

## 7. Create a Cluster Security Group – Protect the Application

### 7.1. Introduction to Security Groups

After configuring the NAT Gateways and Route Tables, I set up Security Groups to control network traffic in RideNow.

A Security Group is an AWS virtual firewall that controls inbound and outbound connections for the resources to which it is attached.
**Inbound Rules** determine which incoming traffic is allowed, while **Outbound Rules** determine which outgoing traffic is allowed.

For RideNow, I create a Security Group named `ridenow-prod-cluster-sg`, intended for use with EKS and application components deployed later.
The goal is to allow resources using the same Security Group to communicate internally without opening direct access from the entire Internet.

### 7.2. Steps to Create the Cluster Security Group

Inbound Rules define which connections are allowed to reach resources associated with a Security Group.
When initially creating this group, I **do not add any Inbound Rules**.
This means that the Security Group does not yet allow new incoming connections from any source under its rules.
The purpose is to restrict unnecessary connections and add only the permissions actually needed by the system.
After creating the Security Group, I add a dedicated rule allowing communication between resources that use this same group.

Outbound Rules define which connections the resources associated with the Security Group may initiate.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cluster-sg-outbound-rules.png)

With this configuration, resources associated with the Security Group can initiate outbound IPv4 traffic using all protocols and ports.
This supports connections to services such as the RDS database or external services through a NAT Gateway.
However, actual connectivity also depends on Route Tables, Network ACLs, and other networking settings.
I retain this Outbound Rule for the Cluster Security Group.

After configuring the Inbound and Outbound Rules, I add a tag for resource management and lookup.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cluster-sg-name-tag.png)

The Name tag helps me identify the Security Group in the AWS Console and query it with the AWS CLI.

### 7.3. Add a Self-referencing Inbound Rule

After creating the Security Group, I add a self-referencing Inbound Rule, also called a **Self-referencing Rule**.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cluster-sg-self-referencing-inbound-rule.png)

Importantly, for Source, I do not choose `0.0.0.0/0`. Instead, I select the Security Group itself: `ridenow-prod-cluster-sg`.
This is called a self-referencing rule because the Security Group refers to itself as the permitted source.

### 7.4. Explain the Self-referencing Rule

A Self-referencing Rule allows resources associated with the same Security Group to send traffic to each other over the protocols and ports specified in the rule.

For example, suppose the system has Application A and Application B, and both use `ridenow-prod-cluster-sg`.
When Application A sends a request to Application B, the self-referencing rule permits that incoming traffic to Application B if the other networking requirements are also satisfied.

Because I select All traffic as the Type, this rule allows all protocols and ports between resources that use the same Security Group.
However, this does not mean that every resource in the VPC can automatically communicate with every other resource. The rule applies only to resources actually associated with the specified Security Group.
It also does not permit the entire Internet to access the application directly.
Therefore, the Self-referencing Rule supports internal communication between application resources while restricting permitted sources.

### 7.5. Results and Next Steps

After deployment and verification, I have created a Cluster Security Group named `ridenow-prod-cluster-sg` in `ridenow-prod-vpc`.
For Inbound Rules, I configured a self-referencing rule that allows resources using the same Security Group to communicate with each other.
For Outbound Rules, I kept the setting that permits all outbound IPv4 traffic to support application connectivity needs.
I also added a Name tag to simplify resource management and AWS CLI queries.
After configuration, I used the AWS Console and AWS CLI on Ubuntu WSL to verify the Security Group ID, Inbound Rules, and Outbound Rules.
The Security Group provides an additional layer of network traffic control for RideNow and prepares the system for deploying application services on EKS.
However, this Security Group has only been created and configured. Its rules take effect on the resources associated with it when those resources are deployed.
**After completing the Cluster Security Group, the next step is to create a Database Security Group** that controls connections to PostgreSQL, allowing TCP port 5432 access only from the previously created Cluster Security Group.

## 8. Create a Database Security Group – Protect the Database

### 8.1. Introduction to the Database Security Group

After creating the Cluster Security Group for the application components, I configure a separate Security Group to protect RideNow's PostgreSQL database.
A Security Group is an AWS virtual firewall that controls incoming and outgoing network traffic for the resources associated with it.

In this section, I create a Database Security Group named `ridenow-prod-db-sg`.
Its purpose is to restrict access to the PostgreSQL database so that only resources associated with the previously created Cluster Security Group can connect over TCP port 5432.
This is PostgreSQL's default port for receiving connections from applications.
Rather than opening the database port to the entire Internet, I use the application's Security Group as the source to improve data protection.

### 8.2. Configure the PostgreSQL Inbound Rule

After entering the basic information, I configure the Inbound Rule.
An Inbound Rule determines which incoming connections are allowed to reach resources associated with a Security Group.
For the Database Security Group, I allow PostgreSQL connections only from the previously created Cluster Security Group.
![RideNow network configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-database-sg-postgresql-inbound-rule.png)

When PostgreSQL is selected as the Type, AWS automatically sets the protocol to TCP and the port to 5432.
The most important setting is Source.
Instead of choosing `Anywhere-IPv4` with the address range `0.0.0.0/0`, I select the Cluster Security Group `ridenow-prod-cluster-sg`.
This means the rule only permits TCP port 5432 connections originating from resources associated with that Cluster Security Group.
Using a Security Group as the Source limits database access and avoids exposing the PostgreSQL port to every IP address on the Internet.
Unlike the Self-referencing Rule in the previous step, I can configure this Inbound Rule when creating the Database Security Group because the Cluster Security Group already exists and has an ID that I can select.

### 8.3. Configure Outbound Rules

After configuring the Inbound Rule, I review the Outbound Rules.
Outbound Rules determine which connections resources associated with the Security Group are allowed to initiate.
According to the RideNow design, I keep the default outbound configuration:
- **Type:** All traffic.
- **Protocol:** All.
- **Port range:** All.
- **Destination:** Anywhere-IPv4, corresponding to `0.0.0.0/0`.
- **Description:** `Allow all egress`.
With this configuration, resources using the Database Security Group can initiate outbound IPv4 traffic according to the Security Group rules.
However, allowing all outbound traffic does not mean that the database becomes directly accessible from the Internet.
Actual access also depends on factors such as subnets, Route Tables, IP addresses, and the RDS service configuration.
In addition, AWS Security Groups are **stateful**, meaning response traffic for an allowed connection is automatically permitted without needing a separate rule for the return path.

### 8.4. Results and Conclusion

After deployment and verification, I have created a Database Security Group named `ridenow-prod-db-sg` in `ridenow-prod-vpc`.
For Inbound Rules, I only allow TCP port 5432 connections from `ridenow-prod-cluster-sg`.
This restricts PostgreSQL database access and avoids opening the database connection port to the entire Internet.
For Outbound Rules, I retain the setting that allows all outbound IPv4 traffic, as specified in the document.
I also added a Name tag to simplify resource management and AWS CLI queries.
After configuration, I used the AWS Console and AWS CLI on Ubuntu WSL to verify the Security Group ID, Inbound Rules, and Outbound Rules.
Creating the Database Security Group adds another layer of network protection for RideNow's PostgreSQL database.
However, at this point, I have only created and configured the Security Group; the RDS PostgreSQL database has not yet been deployed.
In subsequent steps, when creating database instances on AWS RDS, I will associate this Database Security Group with the corresponding instances and use private subnets according to the network design.
**This completes the configuration of the two Security Groups in Section 5.4 Networking: the Cluster Security Group for the application and the Database Security Group for the database.**
Together, these Security Groups establish the foundation for controlling connections between the application and database components once RideNow is deployed on AWS.

## 9. Conclusion

In Section 5.4, I have established the basic networking infrastructure for the RideNow system on AWS.
The system uses one VPC as its private network, divided into two public subnets and two private subnets across two Availability Zones.
The public subnets have an Internet route through the Internet Gateway, while the private subnets can initiate outbound Internet access through NAT Gateways.
I also use Route Tables to direct network traffic and Security Groups to control connections to the application and database.
This network design separates public-facing and internal components and provides a foundation for improving security and availability.
After completing the Networking section, I can proceed with deploying other RideNow components, including the database, cache, and Kubernetes on EKS.
This concludes my presentation on Networking.
