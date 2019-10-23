---
layout: post
title: Site-to-Site VPN using OpenSwan EC2 Instance and Virtual Private Gateway
tags: [AWS, EC2, VPC, VPN, Virtual Private Gateway, Customer Gateway, IPSEC VPN, CLOUD]
---

IPSec (IP Security) Protocol Suite is a set of network security protocols, conisting of different protocols to provide Confidentiality, Integrity, Authentication (CIA) and anti-replay capabilities. IPSEC provides two important capabilities namely `Encryption` and `Authentication` and it can be used in two modes namely `Tunnel` and `Transport`.

`IPSec VPN` works by authenticating and encrypting IP packets from source to destination ensuring security and privacy over Internet. IPsec VPN are mostly utilized in scenarios where two remote locations need to be connected but securely. One use case of `IPSEC VPN` is where a company hosts applications on a cloud network like AWS and wants to enable communication between thier data center and applications running on cloud. All communication data in transit must be confidential and encrypted, hence the use of VPN technology like `IPSEC VPN` is a must.

In this post, I will be going through a step-by-step practical guide on how to create `IPSEC VPN` in tunnel mode between two AWS accounts. One side of the connection will use a OpenSwan Instance(VPC-A) for terminating the VPN connection and the other will terminate at a Virtual Private Gateway(VGW)(VPC-B) .

# Diagram
<img src="/img/architecturevpn1.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 50px;" />

# Steps

### Deploying the EC2 instance

- The first step is to launch a new EC2 instance to run Openswan:
    - Open the AWS console and navigate to EC2 under services.
    - Launch a new EC2 instance.
    - Choose your Linux distribution (In this guide, we will be using the Amazon Linux 2 AMI but Openswan runs on most Linux distributions)
    - Recommended size – m4.large instance. Some developers experienced large amount of packets being dropped with any of the t2 series instances. But for the purpose of this demonstration, I will use t2.micro which falls under free tier.
    - Place the instance in a Public subnet.

- Each EC2 instance performs source/destination checks by default. This means that the instance must be the source or destination of any traffic it sends or receives. However, a VPN instance must be able to send and receive traffic when the source or destination is not itself. Therefore, you must disable source/destination checks on the VPN instance.

- Assign an EIP (Elastic IP) to the Openswan VPN instance.

- Modify the Openswan Security Group
    - Open the necessary inbound ports from your VPC. This will be based on the type of traffic you are sending through the tunnel.
    - Open UDP ports 4500 and UDP port 500 from the remote gateway you are establishing the tunnel with. This will allow the ipsec connection to be established.
<img src="/img/sg.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 30px;" />
```
Here, 13.236.192.60/32 is the public IP of the tunnel on AWS side.
A configuration file can be downloaded from the Site-to-Site VPN AWS Console. This file includes configuration details to be made on the client side (VPC-A) like Tunnel IP Addresses, Pre-shared keys, other configuration details.
```

### Configure the OpenSwan Instance

- Install OpenSwan using the command yum install openswan -y.

- Open /etc/sysctl.conf and ensure that its values match the following,
```
net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.accept_source_route = 0
```

- Apply these changes by executing the command `sysctl -p`.

- Edit the file /etc/ipsec.conf and uncomment the last line include `/etc/ipsec.d/*.conf`.

- Create a VPN configuration file: vi /etc/ipsec.d/{vpnname}.conf and add the tunnel details. An example of this in reference to the diagram above is provided below,
```
conn Tunnel1
authby=secret
auto=start
left=%defaultroute
leftid=13.237.83.228      --- Elastic IP of the OpenSwan Instance
right=13.236.192.60       --- Public IP of the tunnel on AWS Side (VPC-B)
type=tunnel
ikelifetime=8h
keylife=1h
phase2alg=aes128-sha1;modp1024
ike=aes128-sha1;modp1024
keyingtries=%forever
keyexchange=ike
leftsubnet=192.168.99.0/24  --- CIDR Range of VPC-A
rightsubnet=10.0.0.0/16     --- CIDR Range of VPC-B
dpddelay=10
dpdtimeout=30
dpdaction=restart_by_peer
```

- Create a Secrets File: vi /etc/ipsec.d/{vpnname}.secrets and add the pre-shared key for authentication. 
`13.237.83.228 13.236.192.60: PSK "K8WUTRNOVHiCnUae7rjNhMehNgRMjVXub"`
An example of this in reference to the diagram above is provided below,

```
The secrets file is in the format,
{EIP} {Remote Public IP} : PSK “TheSecretPassphraseYouWantToUse”
This information is available from the configuration file downloaded from AWS Console from VPC-B.
```

- Start Openswan:- `service ipsec start`.

- Set Openswan to start when the machine starts: `chkconfig ipsec on`.

- The output of command `ipsec auto --status` is shown below which verifies that the IPSEC tunnel is up.

<img src="/img/tunnelup.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 30px;" />


- Some useful commands to debug IPSEC tunnels,
    - `ipsec auto --up` Tunnel1 to quickly check the status of the tunnel.
    - `service ipsec status` to get generic information about the tunnel. (i.e. if it's running or not)
    - `ipsec auto –status` - This will give detail on the negotiation of each tunnel. You will want to look for the connection name and the phrase “(ISAKMP SA established)”. This will let you know that the tunnel for that connection completed properly. If the tunnel fails to come up this command will also let you know what part of the negotiation failed.

### Configure AWS Virtual Private Gateway on the other side(AWS Account)

#### Customer Gateway (CGW)

- Create a Customer Gateway using internet-routable IP address (static) of the customer gateway's external interface. If the customer gateway is behind a network address translation (NAT) device that's enabled for NAT traversal (NAT-T), use the public IP address of the NAT device. For this example, the customer gateway IP address is `13.237.83.228` which is the Elastic IP of OpenSwan Instance.

- For type of routing, choose static. Usually, in production environments, it is dynamic with BGP.

- For Dynamic routing only, choose Border Gateway Protocol (BGP) Autonomous System Number (ASN) of the customer gateway.

- The screenshot below shows an example configuration of customer gateway in reference to the diagram above,

<img src="/img/cgw.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 30px;" />

#### Virtual Private Gateway (VGW)

- A virtual private gateway is the VPN concentrator on the Amazon side of the Site-to-Site VPN connection. Create a virtual private gateway and attach it to the VPC ( For our example, it is connectivity-vpc) from which you want to create the Site-to-Site VPN connection.

- When creating a virtual private gateway, you can specify the private Autonomous System Number (ASN) for the Amazon side of the gateway. If you don't specify an ASN, the virtual private gateway is created with the default ASN (64512). You cannot change the ASN after you've created the virtual private gateway.

- The screenshot below shows an example configuration of virtual private gateway in reference to the diagram above,

<img src="/img/vgw.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 30px;" />

#### Site-to-Site VPN Connection

- Using previously created CGW and VGW, populate creation of Site-to-Site VPN. Use static routing option for this example. The screenshot below shows an example configuration of Site-to-Site VPN in reference to the diagram above,

<img src="/img/vpnconfig.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 10px;" />

- Then on the Tunnel details page, the status of tunnels will be visible. 

<img src="/img/tunup1.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 30px;" />

- The VPC route table to which the VGW is attached to needs to be updated. In the Route Propagation tab, edit and tick the box that says propagate route. Then, if we check the routes, we will see that the VPN traffic route will be propagated.

<img src="/img/rt.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 30px;" />

#### Verifying the connection

- `VPC-B` has flow logs enabled. We can try using tcptraceroute, tcpdump and VPC Flow Logs to verify connectivity between `VPC-A` and `VPC-B`. Two instances deployed in different AZ's and subnets with private IP addresses allocated to them are used to verify this connectivity.

- Using tcptraceroute and tcpdump,

<img src="/img/vfy1.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 10px;" />

<img src="/img/vfy2.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 30px;" />

- The commands used to get this output are `tcpdump -s 0 -i eth0 -w testcap.pcap` and `tshark -r testcap.pcap | grep http`.

- VPC Flow Logs, in the format,
`<version> <account-id> <interface-id> <srcaddr> <dstaddr> <srcport> <dstport> <protocol> <packets> <bytes> <start> <end> <action> <log-status>`

<img src="/img/flowlogs.png"
     alt="Markdown Monster icon"
     style="float: center; margin-right: 30px;" />

- Related Articles can be found here,

    - [AWS-VPN-DOCS](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html)

    - [AWS-VPN-EXAMPLES](https://docs.aws.amazon.com/vpn/latest/s2svpn/Examples.html)
