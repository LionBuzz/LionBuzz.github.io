---
layout: post
title: Networking Architecture - Hybrid
tags: [AWS, EC2, VPC, VPN, AWS PrivateLink, Transit Gateway, Interface Endpoint, Endpoint Services, CLOUD]
---

Yet another interesting architecture including AWS services like AWS Site-to-Site VPN, PrivateLink and Transit Gateway. This solution enables connectivity between a Corporate Data Center and workloads deployed in AWS Environment.

Let's say we want customer support personnel to connect to their hybrid deployed workloads in AWS from the data center and be able to query, make changes, generate logs, create backups and so on; basically manage the workloads in accordance to business needs. The diagram below provides more insight on this,

<img src="/img/blog.png"
     alt="Markdown Monster icon"
     style="float: left; margin-right: 10px;" />


This scenario can be detailed in the following steps,

* Support personnel will use Site-to-Site VPNs to connect from their data center to the connectivity account which will terminate at the Virtual Private Gateway(VGW) at the AWS side.
* Transit Gateway will have the VPN or VPC attached to it, hence will pick up the connection terminated at the VGW and forward it to the endpoint(AWS PrivateLink).
* An NLB(network load balancer) listening on pre-defined port will pick up the connection on the account where the workloads are.

### Creation of Site-to-Site VPN the following steps,

Customers leverage AWS site-to-site VPN to connect their network to their PCPT AWS Connectivity Account. It comprises of the following components;

* A customer gateway which is a physical device or software application on customer's side of the Site-to-Site VPN connection.
* A virtual private gateway which is the VPN concentrator on the Amazon side of the Site-to-Site VPN connection. You create a virtual private gateway and attach it to the VPC from which you want to create the Site-to-Site VPN connection.
* The target gateway of AWS Site-to-Site VPN connection from a virtual private gateway to a transit gateway. A transit gateway is a transit hub that you can use to interconnect your virtual private clouds (VPC) and on-premises networks. 

### Creation of Transit Gateway involves the following steps,

* Creating the Transit Gateway
* Attaching Your VPCs to Your Transit Gateways
* Adding Routes between the Transit Gateway and your VPCs
* Security groups attached to Transit Gateway must allow ingress and egress traffic on the ports defined.

### Creation of PrivateLink involves the following steps,

* The generation of a DNS hostname, the use of private IP address, the deployment of the endpoint, and its configuration. 
* An endpoint service is created on the Service Provider side which requires an NLB with listeners and target groups configured with right protocol and ports.
* We can specify multiple subnets in different availability zones for high availability and fault tolerance.
* The CIDR Range of the VPC must be allowed in the security group of the target instances for NLB to communicate and perform health checks. 
* An interface endpoint is created on the Service Consumer Side where we lookup for the Service Endpoint using "Service Name" ( Note that the consumer account must be whitelisted in the Service Provider side for Consumer account to find the service and verify it).
* After verifying, the endpoint populates ENI's with private IP addresses in the subnets we specify. 
We can specify multiple subnets in different availability zones for high availability and fault tolerance.
* The IP address of the instances must be allowed in the security group of the ENI's to allow communication between them. 
* The Service Provider account must accept the connection.
* After the consumer creates interface endpoint, the service provider can accept or deny the connection. 
* When an interface endpoint is created, endpoint-specific Domain Name System (DNS) hostnames are generated that can be used to communicate with the service. (For example, vpce-037de48c63vc16c51-4ddrcns3.vpce-svc-ossdec22b247840922.ap-southeast-2.vpce.amazonaws.com). 

### Limitations

* Only TCP traffic is supported for the PrivateLink service.
* Customer needs to define a safe CIDR for the Connectivity Account that is hosting the Production and Non-Production VPCs
* Cross-TGW Security Group Referencing is not supported at launch. Spoke Amazon VPCs cannot reference security groups in other spokes connected to the same AWS Transit Gateway.
* Maximum bandwidth per connection in the transit mesh is limited to 1.25gps. 
* Network Load Balancers(NLB) do not have a security group attached to them. The problem with this is that we have to allow the whole CIDR Range (like 10.0.0.0/16) in the security group of the services ( like EC2 instances) in the Service Provider Account to allow NLB's Target Group to check for health in assigned ports and protocols. On the other hand, in the Service Consumer Account, the Elastic Network Interfaces assigned to each of the subnets we specify have a security group which needs to allow ports and protocols from the services or instances initiating the connection. We can be more specific with this security group by allowing only the instances to communicate with the ENI which then forwards the requests to the NLB and the destination service.

### Note
```
- VPC Peering and Transit Gateway cannot be used with overlapping CIDR Ranges; hence PrivateLink is used in this scenario.
- Also, PrivateLink is uni-directional, so in this scenario, if we need traffic initiated from workloads in AWS environment to reach back to customer's data center, we need a second PrivateLink deployed between Connectivity Account and Customer Deployed Environments(workloads).
```
