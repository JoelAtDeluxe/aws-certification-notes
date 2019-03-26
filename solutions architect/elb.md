# Elastic Load Balancer (ELB)

Splits traffic among instances

## Overview

* Routes requests for a particular service to a set of clustered webservers
  * Tries to distribute the load evenly
  * Runs health checks to determinte load
* Sits in front of the webservers, so dns points to load balancer
  * Naturally, load balancer has the dns name
* Balances within instances in a particular AZ, or across multikple AZ in a single region
* Two parts:
  * ELB Service: this is the amazon service
  * ELB Nodes: A particular running instance of the ELB
* Two Types
  * Classical load balancer
  * Application (layer 7) load balancer (not covered in this course)
    * Container based?
* Classical Load Balancer
  * Supports HTTP(S) / TCP / SSL
    * Does not support http/2
  * Supports ports 1 -> 65535
  * Supports IPv4/IPv6 and dual stack
* ELBs are charged hourly
* ELBs can be deleted
* Deleting ELBs does not remove the registered instances (but traffic is likely not to reach the registered instances)
* Best Practice to delete an ELB
  * Change teh DNS to point to a particular instance, rather than the elb
  * Wait for that change to propagate (route53 takes about 24 hours to propagate)
  * Then delete the ELB
* You can tag ELBs with user data (e.g. "DB ELB")
* In a VPC, ELBs support IPv4 only

## How it works

* EC2 instance must register with the ELB
* Non HTTP/TCP/SSL traffic his to load balancer, but is not passed to balanced instances (the LB reacts to these as if it were the desired computer)
* Responses from an EC2 instance behind a load balancer returns the response to the LB, which passes it the original requester
* Messages _initiated_ from the ec2 instance behind a load balacner do not go through the load balancer (they hit the intended target directly)
* Health checks are conducted to ec2 instances to determinte how busy and healthy they are -- and if they are dead
* Registratino of an ec2 instance may take time
  * ELB validates the instance first
* ELB forwards traffic to eth0 on an ec2 instance
* If using multiple IPs, ELB will forward traffic to the primary IP
* ELBs covering multiple AZ may launch their own ELBs in a particular AZ to better scale to the traffic
  * In order to allow this, you should have *at least* 8 available IP addresses in that subnet (must be size /27 or greater)
  * _For each AZ_

### Listeners

* Processes that anticipate traffic and distribute it accordingly
* Listenrs on the backend do the same thing, for responses from instances

## Health Checks

* Health checks are sent periodically to verify response times
* If a service fails to respond quickly enough, enough times, then it is marked as unhealth
* Unhealthy nodes can be restored to health status. To do this, the ELB will not send new traffic to that service, and instead only send more health checks
* Eventually, the service should recover and be brought back into the ELB
* Health instances are marked as [In-Service]
* unhealthy instances are marked as [Out-of-Service]
* Health checks (via aws console) are conducted via an http ping (not sure what endpoint... `/` ?)
* Health checks (via AWS API) are conducted using a TCP ping
* Http response should respond with a 200 response
* Response timeout is 5 seconds (range is 2-60 seconds)
* Health check period is 30 seconds by default but can be configured (range: 5 - 300 seconds)
* The Unhealthy threshold controls how many messages can be missed before it's flagged as unhealty
  * Default is 2. Range is 2 - 10 missed messages
* Health Threshold controls when an unhealthy instance can be marked as healthy. The instance must respond successfully to multiple consecutive health checks
  * Default is 10. Range is 2- 10 successful messags

