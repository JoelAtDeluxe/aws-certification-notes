# EC2

Elastic Cloud Compute

## Overview

* Virutalized server
* Can create on Shared hardware or a dedicated hardware
  * Shared = shared with other customers
  * Dedicated = hardware is for you only
* Supports Linux (various flavors) or Windows (windows server?)
  * Login via SSH on Linux or RDP on windows
* Resizable capacity
* Users have root access to each ec2 instance
* Can start/restart/reboot/terminate
* SLA 99.95% availability time (Roughly 4 hours per year unavailable / 22 minutes per month)
* Access with key pair (rsa key)
  * User gets the private key
  * AWS/EC2 instance keeps the public key
* Max of 20 EC2 instances per account
  * You can request more from AWS
* Supports two kinds of storage (block store)
  * EBS (elastic block store)
    * Persistant
    * Network attached drives (NAS drives)
    * EBS-backed => has EBS root volume
  * Instance Store (basically tmp?)
    * Non-Persistance
    * Limited to 10GB per device
* EC2 instances are Organized by families
  * General Purpose (balanced memory/cpu, suitable for most situations)
    * e.g. M3/M4/T2
  * Compute Optimized (More CPU than memory)
    * e.g. C2, C4
  * Memory Optimized (more ram)
    * e.g. R3, R4
  * GPU Instances (graphics performance)
    * e.g. G2
  * Storage Optimized (fast storage)
    * e.g. I2, D2
* Spot Instances
  * Can bid on spot instancesa in some situations
* You can encrypt the root volume(partition) (but it's involved), after creation
  * See lecture 63 for a demo
* You can encrypt data volumes during creation
* Each volume can have its own volume type
* The root volume is auto-created on each instance
* Assigned IP Addresses
  * Private is retained over start/stops
  * Public is lost on restart
  * Elastic public addresses are retained

## EBS / Block Store Types

check for updates [here](https://aws.amazon.com/ebs/features/)

* Types
  * General Purpose (SSD backed)
    * Transactional workloads (small dbs, boot volumes, etc. Workloads dependent on IOPS)
    * Sizes between 1TB to 16TB
    * ~10k IOPS~ 16k IOPS
  * Provisioned IOPS
    * Still SSD based
    * Mission critical apps
    * ~32k IOPS~ 64k IOPS (max)
    * 4TB - 16TB
  * Throughput Optimized HDD
    * Ideal for streaming, big data, etc
    * Cannot be used as boot volumes
    * use for frequently accessed or throughput intensive workloads
    * 500 GB - 16 TB
    * 500 IOPS
  * Cold HDD
    * Less frequently accessed workloads
    * Cannot be a boot volume
    * 500 GB - 16 TB
    * 250 IOPS
  * Magnetic EBS (HDD based) -- OLD --
    * Transactional workloads focused on MB/sec (rather than IOPS)
    * 1 GB - 1 TB
* EBS has 99.999% availability
* EBS is _NAS storage_
* You can create point-in-time snapshots of volumes
  * Done manually, or automated via life cycle management
  * Snapshots are stored in S3, but not YOUR s3

### Block Device mapping

* Mapping of blockstores in an AMI
  * Includes both EBS and Instances stores
  * Indicates which stores to use when launching an AMI
  * Viewable, and can view both types of stores (but only EBS from console -- instance storage is querable from instance metadata)
    * curl http://169.254.169.254/latest/meta-data/block-device-mapping/ (from ec2 instance)
* Can add or remove EBS volues whenever
* Cannot add instance volumes, except at create/launch
* For root volumes, you can modify volume size (up only), type, and delete on termination flag

### Optimized Instances

* These enable full use of the EBS volume's provisioned IOPS
* Works for all EBS volume types

#### Single Root I/O Virtualization (SR-I/OV) / Enhanced Networking

* Gets the VNic directly, instead of going through the hyper visor (i.e. the emulation occurs on the vCPU, rather than through a distinct virutalization layer)
* Enables higher network performance (less gets in the way)
* Not available on all instance types
  * Supported on R3/4, X1, P2, C3/4,I2, M4, D2
* Possible to enable on both EBS backed and Instance store instances
* Works across AZ
* Requires an HVM (Hardware Virtual Machine) image (not PV image)
* This is fee-free!

## Placement Groups

Provides enhanced performance among EC2 instances, where they are logically placed closer together

* Logical Grouping (clustering) in the same AZ _or_ different AZ
* Provides low latency and high throughput of inter-instance communication
* Two Strategies
  * Cluster - Clusred into low latency group in single AZ
  * Spread - Across multiple AZ
* No extra cost to create a placement group
* You _should_ use the Enhanced Networking feature
* Try to launch all required instances at the same time (to guarantee availability)
  * There's a possibility that there won't be any more room to enable this if trying to add new instances
  * If this happens, stop/restart all instances
* You can create placement groups via VPC peering
* Cluster Placement Group
  * Single AZ
  * For low network latecy or high network throughput (or both)
  * The majority of the traffic is between instance groups
  * Possible to mix instance types, but you should avoidthis
  * Stopping an instance and restarting may not allow that instance to rejoin the placement group (but it will try to re-join if possible)
  * Cannot span multiple AZ
* Spread Placement Group
  * Launched on different host
  * Multiple AZ 
  * For small number of critical instances that need to be kept separate
  * reduces the risk of simulatenous failure
  * Provides access to distinct hardware
  * Better for mixing instances types
  * MAX of 7 instances per AZ per group
  * Starting an instance group with insufficient unique hardware will fail the request
    * But you can retry later
* Placement groups names must have unique names per account per region
* You cannot merge two placement groups
* You can move instances to a placement group, or move instances between placement groups
* You can remove instances from a placement group
* When editing placement groups, the instance must be stopped before doing anything
* Instances cannot be in multiple placement groups at the same time
* Instances inside Placement Groups can communicate via public or private ip (private ip is preferred)

## Monitoring / Status Checks

* EC2 _Service_ monitors all services and performs Status Checks
  * Status checks happen once per minute
  * Individualized status checks (you only see your own instances)
* Identify hardware/software issues
* Returns Pass/Fail only
* If one or more checks fail, it reports the status as "impaired"
* You can use monitoring services to initiate actions (e.g. reboot) on impared instances
* Once an instance is marked as impared, AWS will schedule a start/stop to move EBS-backed instance to move them to a new host
  * Or you can do this yourself
  * Reboot is (apparently) not the same thing
* EC2 service can send metrics to AWS CloudWatch every 5 minutes (enabled by default)
  * Free of change, called "basic monitoring"
* You cn enable detailed omonitoring on launch (or later) where the ec2 _service_ to send metric data every 1 minute
  * A Fee applies for use (called "detailed monitoring")
* Cloudwatch alarm actions:
  * Stop/Restart/Terminate/Recover
    * Stop/Terminate to save costs (when you no longer need that instance)
    * Reboot/Recover to move instance to new host
  * You can use these to do some automatic maintenance (or other tasks), e.g. you can use this to kill long running processes once they complete

## EC2 Instance States

* Pending (Booting), Running (Available), Stopped, Terminated
* Pictographically:
  * EBS-Backed: [Pending] -> [Running] -> [Stopping] -> [Stopped] -> [Terminated]
  * Instance-Backed: [Pending] -> [Running] -> [Terminated]
* In Pending State
  * Instance gets Private DNS Hotname
  * Maybe Public DNS hotname (if configured to have a public ip)
* When rebooting, state does not change (still [Running])
* Stopping and restarting can cause fees for hourly billed instances
* On Stopped State
  * Retains Instance ID and Root Volume
  * You can detach/re-attach EBS volumes including root volumes
* Instance-back stored instances cannot be stopped, only terminated
* EBS volumes incur charges even when instance is stopped
* EBS volumes remain attached to stopped instances
* Private IPv4 and any IPv6 address is retained
* Instance retains elastic ip address
* If using an ELB, when you stop an instance, you should de-register the instance from the ELB (if stopped for some time)
  * You can re-register with the ELB later
* If using AutoScaling groups, detach instance from auto scaling group (or AS will create a new image)
* Best practice is to use EC2 reboot vs rebooting from the instance
  * EC2 will wait for some time before forcing a hard reboot (in case a normal reboot fails)
  * Cloud watch logs look cleaner
* No charge for instances when in the stopping/stopped/terminated states

### Instance Termination

* EBS _root_ volumes are deleted automatically when instance is terminated
  * Data volumes persist
* You can modify the delete behavior by changing the DeleteOnTermination attirbute of any EBS volume
* i.e. root volumes have the DeleteOnTermination enabled, data values have the flag disabled
* Instance Termination Production
  * Can enable this feature to prevent accidental termination
  * Enable via API, Console, CLI or SDK
  * Can be enabled on either EBS or Instance store instances
  * CloudWatch can only terminate EC2 instances if they do NOT have production enabled
  * You can terminate an instance via shutting down the instance first
  * Can add this at launch, when running, or when stopped
* Immediate Termination Troubleshooting
  * Instance-store based AMI is missing a required part
  * Reached EBS Volume Limit
  * EBS Snapshot is corrupt
  * You can check why via
    * AWS Console: Instances/instance/description tab and look for State Transition Reason
    * AWS CLI: use describe-instance command

### Instance / User Metadata

* data you can use to configure/manage the instance
* examples:
  * IPv6 /IPv6 address 
  * DNS host names
  * AMI-ID
  * Instance-ID
  * instance type
  * local hostname
  * public keys
  * security groups
  * etc
* Metadata can only be viewed from the instance itself
* Metadata is open and can be reviewed by any user on the instance
* Accessible via:
  * GET http://169.254.169.254/latest/meta-data
* To view a parameter:
  * GET http://169.254.169.254/latest/meta-data/<command>
    * e.g. GET http://169.254.169.254/latest/meta-data/host-name
* User Data is a script that is passed when launching the instance
  * Automates part of the (startup?) process 
    * e.g. Updating systems, etc
* User Data is limited to 16KB
* User Data can only be viewed from within the instance
* You can change the user data via stopping the service and going to instance/actions/instance-settings/change user data
* User Data is not protected via encryption

## Migrating VMs to inside AWS

* Usable for VMWare, Microsoft, XEN VMs
  * VMWare -> use VMConnector
* You can export your EC2 images into a format that can be read by one of the above services 
  * Note: You can only do this for instances that were originally imported, not for native EC2 instances
* Works for Windows and Linux platforms
* Can import/export via API or CLI, but not through console
* Instance must be stopped before generating an exportable image

## IAM Role

* Assign IAM role to instances
  * Provide access/privledges to other aws services
  * Has associated policies
* You can edit these policy permissions at launch, or after launch

## EC2 Bastion Host / Jump Host

* Linux: Bastion Host (SSH)
* Windows: Remote Desktop (RDP)
* EC2 Instance with a security group allowing inbound SSH
* Works with either Public or Elastic IPs (prefer Elastic)
* You can further limit which IPs can access the jumphost
* Jumphost allows access to all other services behind it
* You can auto scale bastion host to increase availability
  * Create an Autoscale group with capacity 2, under multiple AZ using an elastic IPs on each

## Purchasing Options

* On-Demand
  * Most Expensive
* Reserved Instance
  * Not launching, but buying capacity
  * done by region/AZ
  * Commit to 1 or 3 year term
  * Can save a large amount of money if you know you will need it
  * When launching an On-Demand instance that matches, you will be charged the reserved price
  * Guarenteed availability
  * Cannot be refunded or cancelled
  * Can resell on the Marketplace (only for AZ scope)
  * Billed no matter what
  * When reserved period ends, on-demand pricing takes over -- does not auto-renew
  * Only applies to On-Demand instances!
  * Scope is tied to region by default, but you can opt to use AZ
  * Can change the scope from AZ to another AZ in the same region
  * You can change the scope from AZ to a region, or a region to an az
  * Can change the instance size to the same family. e.g. m3 XLarge -> m3 Large
    * Smaller size only, but you can get more instances from this (so long as the total fee for the original reservation matches the combined cost of the newer instances)
  * You must submit a request to do these changes
* Spot Instance
  * Bid for capacity
  * Will kill processes when someone outbids you
  * Best for processes that can be done sparely and can be resumed later

## Elastic Network Interface (ENI)

* Eth0 is the primary network interface
  * Cannot move/remove etho0
* You can add more ENI instances (before or after launch)
  * +One at launch
  * +More depending on the size
* ENI is bound to an AZ
* You can specify which IP you want, or Amazon can do it for you
* Security groups applies to the ENI
  * If you have multiple ENIs, they can use separate security groups (not clear on this)
* Attaching an interface while running is referred to as a `hot attach`
* Attaching while stopped is referred to as `warm attach`
* Attaching while stopped is referred to as `cold attach`
* How To
  * Choose "add a device" (on the ec2 instance console?)
  * If you add more than 1, aws will not assign a public ip to eth0. You will need to use an elastic ip
* eth0 is terminated when the instance is terminated
* eth1 is not terminated when the instance is terminated

### ENI IP Addressing

* Each ENI can have:
  * A description
  * A primary ipv4 (private)
  * More secodary ipv4 addresses
  * An elastic IPv4 addresses (via NAT) corresponding to each ipv4
  * A public IPv4 Addresses <-- this exists on eth0
  * 1+ IPv6 address
  * 5 security groups
  * A MAC addresses
  * Source/Destination check flag
* Why Multiple IPs? (instead of multiple ENIs?) (examples)
  * Multiple sites on your instance (multiple SSL certs each associated with one IP)
  * Security/Network appliances in VPC
  * Redirecting internal traffic to a backup ec2 instance (when first is down)
* You can configure secondary IP addresses to be reassigned to another interface
  * Elastic IP will follow this ip
* You can reassign eth0's secondary private IPv4 to another ENI
  * Useful in failure scenarios to redirect traffic
* Private Ipv4 can be associated with only one elastic ip (and viz a viz)
* removing a secondary IPv4 addresses disassociates the elastic IP addresses
* Can remove IP addresses while EC2 instances are running or stopped
* Detaching ENI
  * Other than eth0, any can be detached and reattached to any instance
* Attaching a network interface to another EC2 instance, that instance must be in the same AZ
  * You can even attach in different subnets
* You cannot "team" ENIs

## EC2 NAT Instance Source/Destination Check Flag

* Normally
  * EC2 instances only accept data destined to it
* With a NAT Instance
  * You can enable this functionality via _disabling_ the source/destination check flag
* Flag is enabled by default for any ec2 instances

## Public IP Address assignment

* You cannot control what public ip address you get
* You can enable or disable auto-assign public ip address
* Default value is tied to what the subnet specifies
  * by default, the subnet Default Value is set to enable
  * Custom VPCs have the subnet default value set to disabled

## Amazon Machine Image (AMI)

* Can be generated from a template
* Can be generated from an existing AMI
* Can be custom
* You can share images
* you can make images public so long as it's not encrypted
