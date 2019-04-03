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

## Autoscaling Policies / Plans

* You have separate policies for scale out and scale in
* Scaling Types
  * Manual Scaling ("keep size")
    * Maintain a number of instance at all time
    * Manually change min/max/desired, and attach/detach instances
  * Cyclic Scaling (schedule based)
    * Predictable load change
  * On-Demand / Dynamic (event based) scaling
    * Scaling in response to an event or alarm
* An ASG can have multiple policies attached to it at any time

### Cyclic Scaling

* Use for predictable load change
* Need to configure a scheduled action and capacity change
* Can configure for one-time change, or recuring schedule
* Action must have a unique date/time
* scheduled policies can be edited action after creation
  * Edit with the console or cli

### Dynamic / On-Demand

* Alarm is an object that watches over a _single_ metric (e.g. cput utilization, memory, network in/out)
* "Need" to have both a scale out and scale in policy
* You can use cloud watch monitor and generate alarms
* Simple Scaling
  * Single adjustment (up or down) in reponse to alarm
  * Uses a cool down timer before re-triggering a second time (def. 300 seconds)
    * This is only supported for simple scaling
    * Will ignore all other alarms (scaling activity) until timer expires
* Step Scaling
  * Allows for multiple steps/adjustments to determing scaling
  * Does not wait for a cooldown timer
  * At each step/metric, a certain response is triggered, e.g. a rule might look like this:
    * 60-69% : Remove 5 instances
    * 75-80% : Add 5 instances
    * 80-90% : Add 5 instances
  * Supports a warm up timer
    * ignores alarms until warm up timer expires
    * basically provides a period of time for when to ignore the incoming events, after a responding event
    * Controlled by range 
      * e.g. a rule of 40-49%: +1 would only add a single instance to be launched so long as:
        * The warm up timer is active
        * The next "alarm" that triggers is still in the 40-49% range
      * Once the warm up timer has expired, the step rule can fire again
    * Technically, while the warm up timer is active, the gathered metrics will not include the previous scaling event
* The scaling event cannot add/remove instances above the maximum/minimum number of instances, respectively

### Target Tracking Scaling Policy

* Specifies a single metric to track against. All instances must hit that target, and if not, scaling is adjusted automatically
* analogy: like a thermostat -- Goes up or down to match your desired "temperature"

### Autoscaling Monitoring

* Basic Monitoring - updates every 5 minutes (free of charge) (EC2 instances)
* Detailed monitoring -- updates every 1 minute (for a fee) (EC2 instances)
* When the launch config is done by AWS CLI, detailed monitoring is the default
  * Basic by default when done by the console
* AS service will send aggregate metrics about the group itself
  * Happens every 1 minute by default
* If you make a luanch config change, the changes will only affect new instances
* Please note the following
  * EC2 sends metrics
  * ASGroup gathers metrics
  * The Launmch configu settings must match the ec2 policy settings
  * So, if you want to check scaling metrics every minute, then ec2 should also be set to deliver metrics every 1 minute
  * Likewise, if you only want to check every 5 minutes, then the ec2 instances should deliver metrics every 5 minutes
  * (A mismatch here is basically wasteful)
    * Having the ec2 delivered every minute, with ASG alarm every 5 minuets means that 4/5 ec2 metrics are basically ignored
    * Having an ec2 delivered every 5 minutes, but ASG every 1 minute means that 4/5 ASG checks are basically worthless

