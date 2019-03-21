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
* 

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

## Amazon Machine Image (AMI)

* Can be generated from a template
* Can be generated from an existing AMI
* Can be custom
* You can share images
* you can make images public so long as it's not encrypted
