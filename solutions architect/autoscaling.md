# Autoscaling

Grows/shrinks instances automatically to adjust to demand

## Overview

* Avoids the administrative process of scaling up the platform, then scaling down the platform
* User-defined scaling metrics
* Helps with building fault-tolerant / high availability services
* region specific
* Terminology
  * scale out = growing
  * scale in = shrinking
* Saves costs by cutting down instances when not needed, and scaling out to meet needs
* Components
  * Launch Configuration
    * Config template used to create new ec2 instances for the ASG (autoscaling group)
    * e.g. Parameters
      * Instance Family
      * instance type
      * AMI
      * Key pair
      * block devices
      * security groups
    * Creable via console or CLI
    * Can create a launch config from scratch (you define everything) vs you can use an existing EC2 instance to create the launch configuration
      * AKA: build out a template, or establish a template from a running instance
    * If creating via an existing ec2 instance:
      * Instance AMI must still exist on aws
      * EC2 tags  and additional block store volumes after instance launch, then these will not be used with the template
    * Launch configs are not editable -- if you want to edit, you need to delete and re-create with edits
  * AS Group
    * (Logical) Grouping of EC2 instances
    * Collection of EC2 istances managed by a Auto Scaling Policy
    * Define these metrics:
      * Minimum number
        * Bare minimum to function
      * desired capacity
        * What you want when possible
      * maximum size
        * what we never want to exceed
    * Editable after it is created
  * Scaling Policy
    * Determines when/if and how to ASG scales or shrinks ( ondemand/dynamic scaling, cyclic/scheduled scaling)
* Configurable via Console, cli, sdk, and apis
* Can span multiple AZ
* No cost for using AS Groups (but instance costs still apply)
* Pairs well with ELB, Cloud Watch, and cloud trail
* Tries to have an equal number of instances in all AZs. When not possible, it will try to add/remove new instances on one of the other configured AZs
* Must define a subnet for each ASG in an AZ
* An instance can only belong to a single Auto scaling group

## AZ Balancing / Rebalancing

* If AS is defined on multiple AZs, and the zones are not balanced (equal number of instances on each AZ), then AS will initiate a re-balance effort
  * Goal: Reach an exact/near exact balancing across AZs
  * Launches new instances before removing existing services
  * ex: 6 instances over 2 zones
    * Initially: AZ1: 2 Nodes AZ2: 4 Nodes
    * Goal: AZ1: 3 nodes, AZ2: 3 Nodes
    * Step 1: Try spinning up a new instance on AZ1
      * AZ1: 3 Nodes, AZ2: 4 nodes
    * Step 2: Remove one of the AZ2 nodes
      * AZ1: 3 Nodes, az2: 3 Nodes
  * It will scale up beyond max to reach a proper balance (this is a temporary state)
    * Increase will be `int(max(1, .1*(max_instances)))` (i.e. can grow beyond max by 10%, mininum of 1 extra instance)
    * Avoids performance impact of killing existing 
  * Why would you have an imbalance of nodes?
    * Instances crashing in a particular AZ
    * AZ blows up
    * Updating the AS group to support more (or less) AZs
    * Manually terminating an ec2 instance in a particular AZ
    * No target instances available within a particular AZ for your launch config
    * Spot instances meet your bid price in a particular AZ

## Instance States (add/delete instance to an ASG)

* Conditions
  * Instance must be in a running state
  * AMI must still exist in AWS
  * Instance must not be in another AS Group
  * Instance is in the same AZs as the AS Group
  * ASGroup must have room to include that instance (i.e. ASGroup must not be at max)
* Instance States (during scaling)
  * Pictographically
    ```
    [Pending] -> [InService] ─┬─>  Failed?  ─┬─> [Terminating] -> [Terminated]
                              ├─> Scale In? ─┘
                              ├─> Detatch? -> [Detaching] -> [Detached]
                              └─> [Entering Standby] -> [Stand by] -> [Back to Pending]
                              
    ```
  * Once terminated, EC2 instance cannot be put into the scaling group again
* You can manually detach instances from an ASGroup
* You can manage detatched instance independently or attach to another ASGroup
  * If you remove it from the ASGroup, you can either opt to decrement the ASGroup desired capacity, or leave it.
  * If leaving it, then a new instance will be brought up to restore the one revmoved
* You can put an instance into standby state
  * Health checks are not run in standby states
  * Useful if you want to change the image or troubleshoot issues
* ASGroup can be deleted
  * When deleted, min/max/desired are set to 0, and will thus terminate all EC2 instances
  * If you want to keep the instances, you must detach the instances you want manually

