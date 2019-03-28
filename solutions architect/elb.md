# Elastic Load Balancer (ELB)

Splits traffic among instances

Three Types:

* ALB (Application Load Balancer)
* NLB (Network Load Balancer)
* CLB (Classic Load Balancer) <-- We are covering this one ONLY
  * A combination of ALB and NLB, but old, and likely deprecated

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

## ELB Cross Zone Load Blanacing

* By Default, _ELB_ will balance AZ zones evenly, so zone A will get as many requests as Zone B
* Cross Zone load balacing will instead balace between registered EC2 instances, rather than AZ, for a better overall balacing of load
* Configurable via console, CLI, SDK, and APIs
* ELBs can have distinct names for the account within the aws region
  * Names must be less than 33 characters, with no `-` at the start or end
* ELBs are region specific, but route 53 can be used to balance regions
* ELB needs a subnet to be functional in an AZ
  * Registering a new subnet will mean that the ELB will use that new subnet (and ignore the old one)

## ELB Positioning

* You can use ELBs as either an internet facing tool (to balance traffic from the internet)
* Or you can use them as an internal laod balancing effort for back end services that need balancing
  * e.g. with databases
* If Internet Facing:
  * ELBs nodes will need a public IP address
  * Routes to private ip addresses of EC2 instances
  * One public subnet in each AZ
  * ELB DNS format
    * `(?<name>[a-z0-9]+)-(?<number>\d+).(?<region>.+).elb.amazonaws.com`
    * e.g. `frontface-0123456789.us-west-1.elb.amazonaws.com`
* Internal Load Balancer
  * ELB node s will have a private IP address
  * Routes to private ip addresses of EC2 instances
  * ELB DNS format
    * `internal-(?<name>[a-z0-9]+)-(?<number>\d+).(?<region>.+).elb.amazonaws.com`
    * e.g. `internal-backface-0123456789.us-west-1.elb.amazonaws.com`

## ELB Security

* ELBs must have a security groups
  * You can specify which security group. if you don't, it will take the default VPC security group
    * Default VPC only
  * Must allow health check protocol/ports and listener protocol/ports to reach ec2 instances
* Must ensure that the subnet NACL allows traffic to/from ELB (on front/back end)
* Security Settings
  * Security Group
    * Internet Facing ELB
      * Inbound: Source 0.0.0.0/0 Protocol: TCP port: ELB Listener(http, https, tcp, ssl)
    * internal ELB
      * Inbound: Source VPC CIDR Protocol TCP Port: ELB Listener(http, https, tcp, ssl)
    * Outbound ELB (either):
      * Destination: EC2 Registered Instance Security Group Protocol: TCP Port: Health Check
      * Destination: EC2 Registered Instance Security Group Protocol: TCP Port: Listener
    * Inbound EC2 (behind ELB):
      * Source ELB Security Group Protocol: TCP Port: Listener
      * Source ELB Security Group Protocol: TCP Port: Health Check
    * Outbound EC2:
      * Destination ELB Security Group Protocol: TCP Port: Ephemeral (any)
  * NACL
    * Inbound to ELB Subnet:
      * Source 0.0.0.0/0 Protocol: TCP port: ELB Listener
        * Contact from the outside world
      * Source VPC CIDR Protocol TCP, Port 1024-65535
        * Contact from the EC2 instances behind the ELB
    * Outbout from ELB subnet:
      * Destination: VPC CIDR Protocol TCP, Port ELB Listeners
        * Content sent to Ec2 instances
      * Destination: VPC CIDR Protocol TCP, Port ELB Health Check
        * Health checks sent to ec2 instances
      * Destination: 0.0.0.0/0 Protocol TCP, Port 1024-65535
        * Responses to original requests
    * Inbound to EC2:
      * Source: VPC CIDR Protocol TCP, Port ELB Listeners
        * Inbound requests
      * Source: VPC CIDR Protocol TCP, Port ELB Health Check
        * Inbound health checks
    * Outbound from EC2:
      * Destionation: VPC CIDR Protocol TCP, Port 1024-65535
        * Response to ELB (directed towards outside world)
      * Plus any others for the EC2 instances to reach the outside world directly
    * Settings required when using a non-default nacl
    * If using a default NACL, then all will be allowed inbound/outbound
    * For an internal ELB, replace 0.0.0.0/0 with the VPC CIDR block in all rules

## ELB Listeners

* Listeners can listen on all ports
* ELB supports only layer 4 and layer 7 protocols

### Layer 4 Listeners (TCP/SSL)

* For TCP contenct, message is passed directly
* For SSL content, two scenarios:
  * Option 1: De-encrypt the message and send a TCP message to the service (end-to-end encryption)
  * Option 2: Pass the SSL message directly to the service
* ELB will try to open a connection to the original port indicated
* You can specify the actual source and port by enabling the proxy protocol
  * Note: If there is another proxy ahead of this ELB, then there is a chance to double the number of
    additional headers, which is likely to confused the underlying application. Therefore, only use this if
    no other proxy is ahead of this ELB
  * Likewise, make sure that the underlying application can support the proxied infomration

### Layer 7 Listeners (HTTP/HTTPS)

* For HTTP contenct, message is passed directly
* For HTTPS content, two scenarios:
  * Option 1: De-encrypt the message and send an HTTP message to the service (end-to-end encryption)
  * Option 2: Pass the HTTPS message directly to the service
* You can specify the actual source and port reviewing the x-forwarded-for
* Note, if using HTTPs, you need to have a TLS/SSL certificate
  * This will decrypt the message, then re-encrypt if (if desired) when sending to the ec2 instance

## Sticky Sessions / Session Affinity

* Basically allows sessions over load balancers
  * Not sure how this is really relevant in a REST world
* Will bind the request to a particular ec2 instance
* Not Fault tolerant, obviously
* Requires SSL offloading/termination on ELB
* Cert must be in the same region as the ELB

## Security Policy for SSL/Https

* Componenets
  * SSL/TLS protocols
    * Supports TLS 1.0-1.2 / SSL v3
    * Not Supported: TLS 1.3 / TLS2.0
  * Ciphers (encryption algorithm)
    * Multiple ciphers are supported
  * Server Order Preference
    * Finds the best match on ELB cipher list with the client
      * If enabled, treat server as priority for matches (client must match server) 
      * If disabled, treat the client as prioirty for matches (server must match client)
    * By default, this feature is enabled
  * Certificates include an RSA public key
* ELB support a single X.509 certificate
* If you need to host multiple services, each requiring load balancing and SSL, you'll need to use multiple ELBs (one perr cert)
* Client side certificates are not supported for ELBs (via https)
  * Works via tcp via proxy protocol
    * Not compatible with session affinity (requires https)

## ELB Connection Draining

* Disabled by default
* Provides a wait time for unhealthy instances before terminating all of the sessions on that unhealthy node
  * 300s by default, but configurable 1s - 1h
  * When time elapses, all sessions on that node are terminated
  * If autoscaling is enabled with connection draining, any to-be terminated nodes will be marked for removal, then wait for the connection draining time before killing the server.

## DNS Failover

* You can use route53 to allow for failover between two ELBs in two different regions
* Configurable via route53

## ELB Monitoring

* Monitoring can be achieved via:
  * Cloud Watch
    * Enabled by default
    * ELB sends metrics to cloud watch every minute
    * Only sends metrics if requests are flowing through the ELB
    * Can be used to trigger an SNS message
    * Metrics about ELB instance, not about the requests
  * ELB Access Log
    * Disabled by default
    * Detailed messaging data
    * You can specify an S3 bucket to store the logs (disabled by default as well)
      * No extra charge for enabling the feature, but charged for S3 storage
  * AWS Cloud Trail
    * API Logs for your ELB (i.e. what AWS apis were used with the ELB)
    * Can be stored in an S3 bucket

## ELB Scaling and PreWarming

* ELBs can increase/decrease ELB nodes based on traffic
* Generally takes between 1-7 minutes to detect an increase in traffic and scaple appropriately
* ELBs cannot queue requests
  * So, if too many requests come in, it will return back a 503 error for over-limit requests
* ELB can scale with traffic so long as traffic increases are linear (or step-wise linear)
  * (50% increase every 5 minutes)
* If you need more rapid scaling, then you need to ask AWS to pre-warm ELBs for a particular number of requests
  * Contact with number of expected requests over a particular duration

### Scaling

* DNS records are updated when a new ELB instance is spun up (to help rebalance among the ELB instances)
* ELB uses a DNS records with a TTL of 60 seconds to help onboard/deboarrd ELB instances

## Testing ELB Scaling

* When testing, don't cache dns resolutions
