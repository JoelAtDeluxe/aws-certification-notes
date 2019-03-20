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

## Implied Router

* Central VPC routing function
* Connects different availability zone and connects the vpc to the internet gateway
* Two route tables:
  * Main Route Table
  * Custom Route Table
    * Client controlled
* Routes will have entries to external destinations

### Time Gap -- get from other computer

pass

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

## VPC Direct Connect

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
