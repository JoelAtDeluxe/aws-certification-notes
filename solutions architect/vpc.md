# VPC

Virtual Private Cloud

* Restricted access to other vpc (i.e. explicit approval is required)
* Anything inside the vpc can talk amongst themselves
* VPCs are region specific
* One subnet per availability zone
* VPC per client, or possibly multiple per client (maybe one per department, one per deployment area, etc)
* Client has full control over resources (e.g. compute)
* Analogous to having your own data center, just through AWS
* logically isolated from other vpcs
* One or more ip address subnets per vpc


## Features (components)

* CIDR and IP Address subnets
* IP Address range is not affected by other ips in aws cloud
* Implied router to help communicate between subnets
  * Utilizes route tables
* Internet gateway 
* Security Groups
* Network access control lists
* Virtual Private Gateway

## IPv6 Address

* All are public 

## (implied) Router

* Central VPC routing function
* Connects different availability zone and connects the vpc to the internet gateway
* Two route tables:
  * Main Route Table
    * In a public/private scenario, where subnets are created by the wizard, the main route table will be associated with the PRIVATE subnet
  * Custom Route Table
    * In a public/private scenario, where subnets are created by the wizard, the custom rroute table will be associated with the PUBLIC subnet
    * Client controlled
* Routes will have entries to external destinations

### Route tables

* Route tables apply to subnets
* Each route table will, by default, connect the subnet to the entire vpc (i.e. VPC CIDR block routes to local (i.e. back to itself))
* 50 routes per route table
* 200 route tables per vpc
* Subnets must be associated with only one route table at a time
* Multiple subnets can be associated with the same subnet
* pictographically:
  +----------+
  | subnet 1 |--+
  +----------+  |     +-------------+
                + --> | Route Table |
  +----------+  |     +-------------+
  | subnet 2 |--+
  +----------+

## IP Addresses

** CIDR => IP Address range (x.y.z.0/8, e.g.)
RFC-1918 subnet sizes:

* 10.0.0.0/8
* 172.16.0.0/12
* 192.168.0.0/16
* Private addresses

Extra CIDR block range:

* 100.64.0.0/10

* packets destined to some place outside of the vpc do go outside of the network
* 200 route tables per vpc
* 50 routes per table
* 1000 routes total, I suppose
* Each subnet needs to have exactly 1 route table
  * But, multiple subnets can use the same route table
* By default, aach subnet is associated with the main route table
* You can edit the main route table, but you cannot delete the main route table
* You can reassign the main route table to another routing tbale
* Routing between subnets exists by default (within a vpc)

* You cannot change the CIDR block after the vpc is created
* RFC-1918 IP versioning (192.168.0.0/16 / 172.16.0.0/16 )
* CIDR block range: /28 (16 ip addresses assigned) -> /16 (65536 ip addresses assigned)
* If a different size is needed, you need to make a VPC
* No overlaping vpcs
* You can extend into new CIDR blocks ("secondary block")
  * This will add a route to all routing tables 
  * You can choose something that doesn't overlap with a higher /x number

If you want to expand your cidr block:

| Current block | Permitted block                                                 |
| ------------- | --------------------------------------------------------------- |
| 10.0.0.0/8    | 10.0.0.0/8 (that's not restricted (overlaps?)) OR 100.64.0.0/10 |
| 172.16.0.0/8  | 172.16.0.0/12 (not restricted) OR 100.64.0.0/10                 |

* first 4 and last one are reserved for aws (gateway, multicast?)
  * 10.0.0.0 -> base network
  * 10.0.0.1 -> VPC router
  * 10.0.0.2 -> DNS
  * 10.0.0.3 -> Reserved
  * 10.0.0.255 / Last ip (255 if a /24 -- 127 if /25, etc)
* Internet Gateway
  * One per VPC
  * Gateway connects to other AWS services, internet at large
  * Performs NAT (static one-to-one) between your private and public (or elastic) IPv4 addresses
  * Horizationally scaled, redundant, highly available
  * Supports both ipv4/ipv6



## VPC Types and VPC Security

### Default VPC

* One vpc created in each region
* Default CIDR, Secuirty Group, N ACL, route table settings
* Default Internet Gateway (IG)

### Custom VPC

* AWS owner creates
  * Choice of CIDR block
  * Default security Group, N ACL (Network ACL(access control)), routing tables
  * No IG by default (Still can create one)
* Create via Start vpc wizard
* Tennancy: (Can be on shared hardware (or default?), or dedicated)

### Security Groups (Firewalls at ec2 level)

#### Virtualization Overview

* Host divided into virtualization layers  (AWS is based off of x86 architecture)
  * Has physical components (CPU, Memory, NIC, Storage)
* On top of this, you get a virtualization layer
  * ⬆️ You get virtual components from above
  * ⬆️ OS is built on top of the virtual components (These are EC2 instances)

From the VNIC/ENI (virtual Network interface card / Elastic Network Interface)

* Security groups formed at the ENI/VNIC level

+------+
|      |=----------+   <--- IN BOUND ---  (From the internet : We care about where is it coming from (i.e. which security group)?)
| EC2  |  VNIC/ENI |
|      |=----------+   ---- OUT BOUND --> (To The internet : We care about where is it going (i.e. which CIDR block)?)
+------+

#### Security Groups, revisited

* Security Group is a virutal Firewall
* Controls traffic at the VNIC level
* AWS allows 5 security groups at the same EC2 instance
* Security groups are *stateful* (If in bound was allowed, then out bound is (always) allowed, for that inbound request)
* Can only define allow rules (no deny rules are permitted) (implicitly denied) AKA, This is a whitelist
* Security groups are associated with EC2 instances
* All rules are evaluated to find a permitted rule (else denied)
* You cannot delete the default security group
* Security groups belong to VPCs, so can be assigned to multiple Availability Zones
* Changes to security groups are immediate (same applies to NACL)

##### Default vs Non-Default Security Gruops

* Default (initially):
  * Inbound rules allows instances of the same security group to talk to each other
    * (Source of the rule indicates the security group)
  * All Outbound traffic is allowed
* Non Default (initially):
  * No Inbound rules: All traffic is denied
  * All Outbound traffic is allowed

* Subnet uses NACL to secure vpc
* EC2 uses security groupes to secure itself

    Out Bound:
    EC2 --> Security Group --> NaCl --> Subnet --> Internet
    In Bound:
    Internet --> Subnet --> NACL --> Security Group --> EC2

### NACL / Network ACL / Network Access Control Lists

* NACLs are stateless (For Outbound traffic, there must be explicit rules to allow it, even if the message is in response to a previously allowed inbound traffic event)
* NACL supports BOTH permit *and* deny rules
* NACL is a set of rules with a number (evaluated in order from lowest to highest)
  * Rule spacing is arbitrary per customer choice
  * Leave sapce to allow for future edits
* NACLs ends with explicit DENY
* subnets must be associated a NACL. IF not specified, they are associated with default NACLs
* Default NACL Rules (initially)
  * Allows All Inbound Traffic
  * Allows All Outbound traffic
* Non-Default NACL Rules (initially)
  * Denies all inbound traffic
  * Denies all outbouund traffic

### Security In General

* In addition to NACLs and Security Groups, you can also run your own firewall software at the EC2 instance level (after SecGrp)
* All changes in the NACL/SecGrp take effect immediately

### NAT Instances

* Nat Instance is simply an EC2 pre-loaded with a NATing image (allows routing of messages)
  * Allows private subnet EC2 instances to contact internet
* Would use this in situations where you have some routes that are private (e.g. a private subnet) that occasionally need to contact the outside world (e.g. for security patches)
  * To do the above:
    * Pictographically [PrivateEC2] --> [OUTBOUND_PrivateSG] --> [INBOUND_PublicSG] --> [NAT_Instance] --> [OUTBOUND_PublicSG] --> [Internet]
      * Requires:
        * Private SG to allow OUTBOUND traffic to NAT/NatSG
        * NATSG to allow INBOUND traffic from PrivateSG
        * Optionally allow (e.g. IT Administration, with additional rules)
          * NATSG to allow INBOUND traffic from Inet
          * NATSG to allow OUTBOUND traffic to PrivateSG
          * Private SG to allow INBOUND traffic from NATSG
* Nat instances need to be assigned a SecGrp
  * SecGrp needs to allow OUT BOUND http/https (80/443) traffic
  * SecGrp probably needs to allow IN BOUND SSH (22) traffic (to allow IT administration)
* Public EC2 instances do not require this NAT

## VPC Peering

Allows VPCs ~in the same region~ (regardless of owner) to communicate

* Can be established on ec2 or subnet level
* CIDR blocks cannot overlap between VPC A and B
* VPCs peering is encrypted and runs on the AWS backbone, not over the general internet
* B must accept A's request to establish a link
* VPC peering is already fault tolerant, and is otherwise completely managed by aws
  * No single point of failure
  * No bandwidth bottleneck
* Why use one?
  * data/resource sharing
  * Merge accounts
* Routing tables need to be updated to reach the other VPC
* If there are 3 VPCs, with connections A <-> B and A <-> C: !(B <-> C) (no transitive connections/no edge connections)
* There is a limit of 50 VPC active pairing and 25 Pending peering connections
* Placement groups can span VPC peering

## VPN

* VPN = Virtual Private Network
* Goes over the internet
* Cheap, quick, easy
* VGW (virtual gateway?) is required on VPC side, and customer gateway on client side (data centers)
* (redundant) Tunnels pairs are established for each VPN connection
* Downside: has to deal with internet latency
* You cannot use the NAT gateway to get to internet
* Use dynamic route propagation to link customer gateway subnets and VGW 

### VPN Cloud Hub

* I have no idea what this guy is talking about

## VPC Direct Connect (DX)

* High-performance/low latency connection to aws
  * Similar to something like aspera?
  * Requires physical infrastructure(?) to link HQ/DataCenter to AWS
* Need to request from AWS -- process takes weeks/months
* This would be a higher performance option compared to VPN
* Works via 802.1Q Protocol (VLAN)
  * VIF (virtual InterFace)
* Rates: 50Mbps -> 10+Gbps
* Requires 2 VIFs
  * public vif to reach (public) AWS service (use public routable ip addresses)
  * private vif to reach private VPC (can use private routable ip addresses)
    * Seems to use 169.254.X.X ip addresses
* Cannot establish layer 2 over DX (must be layer 3) (not sure what this means)
* You cannot use a NAT gateway in the vpc over DX connection
* Pictographically:
  * [HQ/DataCenter] <-> [CustomerRouter] <-> [DX_Connection] <-> [AWS_DX_Location]
* Uses eBGP routing (what is this?)
* DX connection have access to all AZs in a region
* Can establish a IPSec VPN tunnel to contact remote regions as well (over public VIF)

### High Availability with DX

* Use multiple customer routers each with a direct connection (1 router per DX connection -- probably only need 2?)
  * One active, one passive (for redundency situations)
* This is an expensive redundency
* Pictographically
  * Active: [HQ/DataCenter] <-> [CustomerRouter] <-> [DX_Connection] <-> [AWS_DX_Location]
  * Passive: [HQ/DataCenter] <-> [CustomerRouter] <-> [DX_Connection] <-> [AWS_DX_Location]

Alternatively:

* High Availability with Fault Tolerence
* One DX connection, one passive VPN connection
* Pictographically:
  * Active: [HQ/DataCenter] <-> [CustomerRouter] <-> [DX_Connection] <-> [AWS_DX_Location]
  * Passive: [HQ/DataCenter] <-> [CustomerRouter] <-> [VPN_Connection] <-> [AWS_VPC]

## Transit VPC/ Private End points

### Transit VPC

* Allows communication between peered networks that would otherwise be unable to communicate
* Not an explicit AWS service
* Essentially an EC2 instance on the VPC that can proxy events from one service to another
* Not managed, not high availability -- you need to manage yourself

### VPC Endpoints

* Gateways from VPC to other AWS services without going through internet
* Go through trouter to VPC endpoint to aws service
* Connects via private IP addresses
* Will allow services/apps to not need to use public ip addresses to communicate with each other
* Endpoints are redundant, scaled, highly availabile
* Two types of endpoints, depending on service
  * Interface Endpoints
  * Gateway endpoints (S3/Dynamo DB)
    * Reqauires endpoint policy control to control access
    * Endpoints only works for same region only

## VPC Flow Logs / DHCP Option Sets

### Flow Logs

* They're logs
  * they capture IP traffic 
* Logs at VPC, subnet, or network interface
* At VPC/Subnet level captures each network interface under it
* Can be published to cloudwatch or S3
* To Create one:
  * Specify the resource to capture
  * The type of traffic to capture (accepted traffic, rejected, or all)
  * The destination to store the logs
* Cloud Watch log charges apply regardless of destination
* Logs are generated after several minutes

### DHCP Option Sets

* You do not need to use AWS DNS for your VPC
* You cannot use Route53 for your on-prem infrastructure
* You cannot modify DHCP options
  * You can create or delete them
  * you can have multiple, but only one active at a time
* DHCP Option sets effects running instances, but effects propagate after some time (hours)

## Problem Hints

* If A cannot talk to B
  * Rephrased: Some Message _outbound_ from A, going _inbound_ to B, does not reach B
  * Pictographically (message originates from [A]):
    * In Request
      * [A] -> [OutBound_SecGroup] -> [OutBound_NACL] -> [NO_ISSUE_Routing] -> [Inbound_NACL] -> [Inbound_SecGroup] -> [B]
    * In repsonse (i.e. if message from A does hit B):
      * [B] -> [NO_ISSUE_OutBound_SecGroup] -> [OutBound_NACL] -> [NO_ISSUE_Routing] -> [Inbound_NACL] -> [NO_ISSUE_Inbound_SecGroup] -> [A]
  * Must be a security related issue, so it could be a NACL or Security groups
    * Therefore issue cannot be with routing
    * Remember that these are directional services, and therefore you need to look at inbound/outbound rules
      * In Request:
        * To Check: Can A send an out bound message to B
          * NACL _and_ Security Group on Source Instance (A)
        * To Check: Can B receive an in bound message from A
          * NACL _and_ Security Group on Destination Instance (B)
      * In Reponse:
        * To Check: Can B send an out bound message to A
          * On NACL _only_
        * To Check: Can A recieve an In Bound message from B
          * On NACL _only_
* InBound vs Outbound
  * On NACL:
    * Inbound: From the outside world to the subnet (or across subnets)
    * Outbout: From the subnet to the outside world (or across subnets)
  * On Security Groups:
    * Inbound: From (ENI/VNic) to EC2
    * Outbound: From EC2 to ENI
* NACL vs SecGrp
  * Choose NACL if you:
    * Want to implement Explicit Deny rules (e.g. to prevent malicious users)
    * Want to reduce traffic to your SecGrp (and therefore EC2 instances)
