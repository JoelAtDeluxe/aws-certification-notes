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

## ELB and Autoscaling

* ELB and AS Group must be in the same group
* Instances in the ASGroup will automatically registered with the ELB
  * Uses the ELB defined in the ASGroup
* Instance and ELB must be in the same VPC
* Detaching the instance will dereigster it from the ELB

## Auto scaling group Health Checks

* AS by default uses EC2 status checks to determine health of the instance
* When you have 1+ ELBs defined with an ASG then you can configure AS to use the ELB health checks as well
* Instance have a grace period to "warm up" before a health check is triggered, started via launch
  * By default, a new instance is given 300 seconds for the grace period
  * You can set the grace period to 0, effectively disabling it
* Once a soruce is marked as unhleathy (from any source), then the AS will replace the isntance
  * Once it has been marked as unhealthy, it can't _automatically_ recover from it's unhealthy designation
  * While an instance has been declared unhealthy, there is a short window of time before termination where you can mark the instance as healthy (use as-set-instance-health)
  * Better to use standby mode if you want to do any edits to a running instance
* Unlike AZ Rebalacing, termination of unhealthy instance will destroy unhealthy instance first, then replace
* If EBS volumes or elastic IPs are attached to a terminated instance, they will become detached, and need to be manually re-attached to new intances

## Spot instances with AS Group & SNS

* Cannot mix spot ASG and on-demand ASG
* Bid price is set for all (specified) AZs 
* AZs will still rebalance, provided that the AZ bid price is higher than the market price
* (Note: you will need to create a new launch configuration if you want to increase the bid price)
* You can configure Auto Scaling to send SNS email notifications when:
  * Instance launches or fails to launch
  * Instance terminates or fails to terminate
* You can merge ASGroups via the CLI
  * This can turn a 2+ ASG in distinct AZs into a single Multi-AZ ASG
  * Once this is done, you can delete the older ASGs, as they won't be needed
  * (you have to rezone one of the groups to cover the others)
  * You have to merge two into an existing autoscaling group -- creating a new one will not work
* 