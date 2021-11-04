# S3
- By default all new buckets are private
- You can have bucket policies and ACLs (ACL can go down to individual objects)
- Read after write consistency for puts of new objects
- Eventual consistency for overwrite PUTs and DELETEs
- Storage classes
    * S3 standard (99.99 availability, 11 9s durability) can sustain loss of 2 facilities concurrently
    * S3-IA lower fee than S3 but you are charged a retrieval fee (99.9% availability)
    * S3 One Zone - IA lower cost option if you don't need multi AZ data resilience (99.5% availability)
    * S3 Intelligent Tiering
    * S3 Glacier
    * S3 Glacier Deep Archive (12 hours of retrieval time, lowest cost)
    * Reduced Redundancy storage (99.99% durability and 99.99% availability)
    * S3 intelligent tiering 99.9% availability
- Encryption
    * SS3-S3: S3 Managed keys (AES-256)
    * SSE-KMS: AWS KMS managed keys
    * SSE-C: server side encryption with customer provided keys
    * You can also encrypt objects yourself, this is called client side encryption
    * For encryption: x-amz-server-side-encryption: AES256 (sse-s3 managed keys) or ams:kms (KMS managed keys)
    * You can create a bucket policy which denies any S3 PUT requests which do not include the x-amz-server-side-encryption parameter in the header
- Cross region replication
    * Versioning must be enabled on both source and destination
    * Regions must be unique (cross region)
    * Files in an existing bucket are not replicated automatically
    * Delete markers are not replicated
    * Deleting individual versions or delete markers will not be replicated
- S3 transfer acceleration is achieved via Edge locations
- CloudFront origin can be S3 bucket, ec2 instance, ELB or Route53 or custom origin (non-AWS)
    * Web distribution for general purpose web apps
    * RTMP for media streaming
    * If you restrict s3 bucket access (only serve via CloudFront) then you need an origin access identity
    * Signed url or signed cookie access: imagine you have paid content
    * Can enable WAF for attacks
    * Can use own certificate for ssl or use one from ACM or IAM
- http://s3-eu-west-1.amazonaws.com/mybucket
- http://mybucket.s3-eu-west-1.amazonaws.com
- Minimum billable object size 128KB for S3-IA
- Max 5 TB files
- We can enable object-level API activity logging via CloudTrail for additional cost
- We can enable CloudWatch request metrics for a bucket for an additional cost
- Public access can be enabled/disabled at the account level as well as bucket level
- You can't upload publicly accessible file to S3 (uses ACL) in a private bucket, it gives an error
- Headers for PUT: Host, Content-Type, Content-Length, x-amz-meta-author etc.
- An object has
    * key
    * value (data)
    * version id
    * metadata
    * subresources
        - bucket policies, acls
        - cors
        - transfer acceleration

## CORS
- Static web hosting url: http://mybucket.s3-website.eu-west-1.amazonaws.com
- CORS Access-Control-Allow-Origin header should be present for CORS
- If bucket A loads a file from bucket B, then we need to enable CORS on bucket B!

# Storage gateways (S3)
- hybrid cloud storage
- supports both object and block based storage
- provides low latency by caching on-premises
- has three storage interfaces, tape file and volume
- you use a virtual appliance gateway or purchase hardware appliance

## File
- NFS/SMB to store/retrieve files in S3
- Recently used data is cached on the gateway

## Volume
- iSCSI, data is stored in S3
- To access the volumes on AWS, take EBS snapshots and create EBS volumes from them (point-in-time snapshots are possible)
- Has two flavours: cached or stored
    * cached: primary data is written to S3, retain frequently accessed data locally
    * stored: primary data is stored locally, entire dataset is available for low-latency access while asynchronously backed up to AWS.

## Tape
- iSCSI virtual tape library VTL, stored in S3 as well

# EC2
- You can attach detach roles on running ec2 instances and it takes effect immediately

## EBS
- 5 types
    * gp2- general purpose (SSD) - most workloads, less than 10K IOPS
    * io1 - provisioned IOPS (SSD) - databases
    * st1 - throughput optimized hdd (magnetic) - big data & data warehouse
    * sc1 - cold hdd (magnetic) - lowest cost
    * standard - magnetic - previous generation
- ebs volume snapshots are incremental (only blocks that change are moved to S3)
- To snapshot root ebs volume, you should stop the instance before snapshot to make sure it's consistent
- You can create AMI from both volume and snapshot
- You can change EBS volume size on the fly
- EBS volumes must be in the same AZ as the ec2 instance
- To move an EC2 volume from an AZ to another
    * take a snapshot
    * create AMI from snapshot
    * use the AMI to launch the EC2 instance into new AZ
- Placed in a specific AZ and automatically replicated to protect from failure of single component (spread across AZ)
- Provisioned iops (io1) can do more than 10K IOPS, up to 20K IOPS per volume
- st1 & sc1 (cold) cant be a boot volume
- Magnetic (standard) can be boot volume
- Spot, if you terminate spot you will be charged the complete hour, if AWS terminates it you are not charged for partial hour

## Encryption
- Snapshots of encrypted volumes are encrypted automatically
- Volumes restored from encrypted snapshots are also encrypted automatically
- You can share snapshots but only if they are unencrypted (can share with other accounts or make public)
- You can encrypt root device volumes upon creation of the instance
- ebs root volumes of default AMI's cant be encrypted
- To encrypt root volume, do it while creating the AMI
- Additional volumes can be encrypted (not the root)
- To encrypt root volume of a running instance:
    * create snapshot of unencrypted root device volume
    * create a copy of the snapshot and select encrypt option
    * create AMI from encrypted snapshot
    * use the AMI to launch encrypted instances

## EBS vs instance store
- Instance store is called ephemeral storage
- Instance store volumes cant be stopped, if underlying host fails you lose all the data
- EBS backed instances can be stopped and you dont lose data
- You can reboot both, you wont lose data in any
- By default root volumes will be deleted on termination by default
- With ebs volumes we can tell AWS to keep the root device volume

## EFS
- You can't have two ec2 instances share EBS, but you can with EFS
- Storage capacity is elastic
- Supports NFSv4
- Pay for storage you use, preprovisioning is not required, pay as you go
- Supports 1000s of concurrent nfs connections
- Read after write consistency
- Data is stored across multiple AZs within a region

## Placement groups
- Name must be unique within your AWS account
- Clustered, spread, partitioned
- Clustered is within single AZ, recommended for apps that need low network latency and high throughput
- Only certain instances can be launched into clustered groups
- Spread: each instance placed on distinct underlying hardware, good for small # of critical instances that should be kept separate from each other
    * think of individual instances
    * supports a maximum of seven running instances per Availability Zone
- Partition: partition within a placement group has its own set of racks
    * no two partitions share the same racks
    * think of multiple instances
- You can't move existing instances into a placement group
- You can't merge placement groups

# Databases
## RDS
- OLTP: SQL, MySQL, PostgreSQL, Oracle, Aurora, MariaDB (dynamodb nosql)
- OLAP: Red Shift (data warehousing)
- Elasticache: Redis, memcached
- Read replica
    * used for performance of reads (to scale for read heavy workloads)
    * async replication from primary RDS instance
    * up to 5 read replicas of any database
    * you can have read replicas of read replicas (watch out for replication latency, it takes time for data to be replicated)
    * must have automatic backups turned on in order to deploy a read replica
    * each replica will have its own dns endpoint
    * you can have replicas that have multi AZ
    * you can create read replicas of multi-az source databases
    * replicas can be promoted to be their own databases (this breaks replication of course)
    * you can have a replica in a second region
    * not available for sql server or oracle
    * you can enable encryption on read replica even if primary is not encrypted
    * you can create read replica of multi AZ primary database
- Backups
    * automated (in scheduled maintenance windows)
    * database snapshot (manual)
    * snapshots are always stored even after you delete the RDS instance, whereas automated backups are deleted
    * automated backups take full daily snapshot and also store transaction logs so you can turn to any point in time (down to a second)
    * automated backups retention period is between 1-35 days
    * automated backups enabled by default
    * backup data is stored on s3 and you get free storage equal to size of the database
    * when you restore either from automated backup or snapshot, it will have a new dns endpoint
- Encryption
    * you cant encrypt existing DB instances, to do that you first create snapshot and make a copy of the snapshot and encrypt the copy

- MultiAZ
    * used for disaster recovery
    * you can force a fail-over from one AZ to another by rebooting the RDS instance
    * Synchronous replication
    * you don't need to enable multi AZ for aurora as it's by default multi AZ
- Aurora does not have free tier yet
- You attach a security group to RDS instance
- To connect from a web server to the RDS instance, add a rule to security group of the RDS instance for inbound 3306 from WebServer security group

## DynamoDB
- Stored on ssd
- Spread across 3 geographically distinct data centers
- Eventual consistent reads by default and you can have strongly consistent reads

## Redshift
- Only available in 1 AZ
- Backup enabled by default with 1 day retention period
- Max retention period is 35 days
- Always attempts to maintain at least three copies of the data (original and replica on compute nodes and a backup on S3)
- Can replicate async snapshots to S3 in another region for disaster recovery

## Aurora
- 2 copies of data is contained in each AZ with min 3 AZs. So that makes 6 copies of the data.
- Share aurora snapshots with other AWS accounts
- 2 types replicas available:
    * Aurora replicas (automated failover is only available with Aurora replicas)
    * mysql replicas (no automated failover)

## Elasticache
- Useful for read-heavy operations with data not changing much
- Redis is multi AZ with failover and persistence
- You can do back ups and restores of Redis
- If you need horizontal scaling, use memcached (which is also simpler to use)
- Memcached is not persistent and not multiAZ
- For multi-threading and large cache nodes, use memcached
- Redis can do sorted sets leader boards, pub/sub and complicated data structures
- Two caching strategies
    * lazy loading: loads data to cache only when necessary
                    if data not found in cache, returns null so you need to
                    fetch the data from the db and write to the cache
    * write through: whenever data is written to database, it's written to cache
                    involves a write penalty
                    never get stale data
                    if node fails, data won't be available until it's written or updated
                    you end up with waste of resources if most data is not read

# Route53
- Maps domains to EC2s, ELBs, S3 buckets etc.
- ELBs do not have predefined IPv4 addresses, resolve to them via DNS name
- A record: peoples name and tel number
- CNAME: batman: see bruce wayne
- Alias record: can map zone apex to an s3 bucket (cname can't do this) (zone apex = naked domain)
- Records:
    * SOA: start of authority
    * NS: name server
    * A records
    * CNAMEs
    * MX Records
    * PTR Records
- Routing policies:
    * simple (only one record with multiple IP addresses, can't do health checks, returns IPs randomly )
    * weighted (20%, 80% etc)
    * latency based
    * fail-over (uses health check, active/passive)
    * geolocation
    * geoproximity (traffic flow only)
    * multivalue answer (same as simple but you get health checks)
- Health checks can be set on individual record sets

# VPC
- A VPC belongs to a region
- Largest ip range is /16 (can't have /8 for instance) => we usually use 10.0.0.0/16
- If you select dedicated tenancy, it's very expensive but you get dedicated hardware
- When you create a new VPC
    * - no subnets are created
    * + a default route table is created
    * - no igw
    * + default nacl is created
    * + default security group is created
- Base + 2 is reserved for DNS server
- Traffic goes through router => nacl => security group
- You can have multiple CIDR blocks in a VPC
- Route table belongs to the VPC
- You can associate subnets to route tables, if not they are by default associated to the main route table
- Any subnet created in the VPC is automatically able to talk to each other (there's an entry for this in the main route table)
- Always keep the main route table private (don't add a route out to the igw in there) instead create another route table for public access and only associate the public subnet with it

## Subnet
- A subnet lives in a specific AZ, can't span multiple AZs
- You can have more than 1 subnet in an AZ
- First 4 and last IP are reserved in each subnet
    .0 is the network address
    .1 is for the vpc itself
    .2
    .3 for future use
    .255 network broadcast
- You select a CIDR block as well (like in VPC)
- Public subnet:
    * you can select "assign auto public ip address" for a public subnet
    * create igw and attach to vpc
    * add a route to igw in the route table associated with this subnet
- A subnet can only be associated with one route table at a time

## Security Groups
- Stateful
- Do not span VPCs, you have to create for each VPC separately
- All inbound traffic is blocked by default
- All outbound traffic is allowed
- You can allow rules but you can't deny rules! (if you don't specifically allow a port it's denied by default)

## NACL
- Stateless
- Default NACL allows everything
- When a new NACL is created, it denies everything by default
- Attaches to a subnet
- Rules are executed in order, once a rule fits it does not look at other rules
- A subnet can be associated to 1 NACL at any time
- If you don't associate a subnet with a NACL, it's associated with the default NACL
- You can block specific IPs with NACL but not with security groups

## IGW
- Allowed 1 per VPC
- Allows VPC to talk to the internet

## NAT
- Needed to allow private subnets to talk to outside world (for software updates for instance)
- Sits in the public subnet
- For NAT instances (EC2)
    * you should disable source/destination check
    * add a route in the private subnet's route table destined to the internet and the target should be the NAT instance
- use a NAT gateway, better availability! redundant inside the AZ
    * not associated with a security group!
    * 1 NAT Gateway in one AZ, can't span multiple AZs (create one for each AZ for availability)
    * NAT gateway is created in a subnet, choose the public subnet for it
    * automatically creates an Elastic IP and attaches to it in the process
    * Add a route in the private subnet's route table destined to the internet and the target should be the NAT Gateway

## VPN
    * Customer gateway (anchor on client side of a VPN connection)
    * Virtual private gateway (anchor on AWS side of a VPN connection, provides two VPN endpoints - tunnels for automatic fail-over)
- 5 VPG per account per region
- 50 CGW per account per region
- 10 IPsec VPN connections per VPG
- CGW can be software or hardware appliance

## VPC Endpoints
- Interface endpoint or gateway endpoint
- Interface endpoint: almost all services (SQS, Kinesis etc.)
- Gateway endpoint: S3 and Dynamodb
- Does not require direct connect, NAT, VPN or internet gateway

## Bastion
- For SSH!
- Can't use NAT Gateway as a bastion host

## Peering
- Not transitive, A-B and B-C peer does not allow A to C
- Can peer across regions
- Does not support edge to edge routing

## Flow logs
- VPC, Subnet and Network Interface level
- Can't enable flow logs with peer VPC if that VPC is not in your account

## Security
- AWS Shield operates on layer 3-4 to protect against DDoS attacks
- AWS WAF can protect from cross-site scripting, block IP addresses based on rules etc.

# High Availability
- 3 types:
    * Classic
    * Application (http/https) can do path patterns
    * Network load-balancers (for performance, can do layer 4)
- 504 error means application is not responding within the idle timeout period, not a problem with the LB itself.
- X-Forwarded-For header carries end user's IP
- Instances monitored by ELB are reported as InService, OutofService
- Cross zone load balancing enables to balance load across multiple AZs.
- Classic only uses round-robin for TCP listeners
- ALB selects a target based on routing rules and then uses RoundRobin to select a node
- NLB does not use RoundRobin
- With classic load balancer, cross-zone load balancing is an option
- ALB comes with cross-zone load balancing enabled by default
- At least two subnets for the load balancer!
- we also assign a security group to an ELB
- ALB has targets, you can create a target group for each micro-service for instance
- ALB layer 7
- NLB layer 4 (expensive)
- Classic, both layer 7 and 4 but layer 7 is not as intelligent as ALB's (just sticky sessions and x-forwarded-for)

# Support
- Basic
- Developer (12-24 hours at local business hours)
- Business (1 hour SLA for urgent)
- Enterprise (15 min SLA)

# Serverless

## Lambda
- Runtimes
    * nodejs
    * python
    * ruby
    * go
    * .net
    * java
- Billed per
    * \# of requests (first 1 million request free, then 0.2$ per million request)
    * duration (depends on the memory as well, GB-second)
- Each lambda version has a unique ARN
- After you publish a version, it's immutable
- $LATEST is always the latest version of lambda
- Sample qualified ARN: arn:aws:lambda:eu-west-1:123456789:function:helloworld:$LATEST
- Unqualified ARN: without the version at the end
- You can create aliases to point to specific versions
- Only the latest version can be edited, once you publish a version you can't alter the code (immutable)
- You can split traffic between lambda functions via alias (blue/green deployments, a/b testing)
- You can't split traffic for $LATEST, instead create an alias to $LATEST first and then use that to split
- Lambda triggers:
    * Services that lambda reads events from:
        - Kinesis
        - Dynamodb
        - SQS
    * Services that invoke lambda synchronously
        - ELB
        - Cognito
        - Lex
        - Alexa
        - API Gateway
        - CloudFront
        - Kinesis Data Firehose
    * Async
        - S3
        - SNS
        - SES
        - CloudFormation
        - CodeCommit
        - Config
        - CloudWatch logs & events

## X-Ray
- Application integrates with X-Ray SDK, X-Ray daemon runs on the server and listens on UDP, then calls the X-Ray API
- Integrates with ELB, Lambda, API Gateway, EC2, Elastic Beanstalk
- Supports Java, Go, Nodejs, Python, Ruby, .Net
- The SDK provides
    * interceptors to add to your code to trace incoming HTTP requests
    * Client handlers to instrument AWS SDK clients that your app uses to call other AWS services
    * An HTTP client to use to instrument external HTTP calls to custom services

## API Gateway
- Can do caching
- Can integrate with lambda, EC2 or Dynamodb
- Exposes https endpoints
- You can import swagger v2.0 definition files
- By default API Gateway limits steady-state rate to 10K TPS
- Max concurrent requests is 5000 across all APIs within an AWS account
- If you hit more than that, you'll get 429 Too Many Request error
- You can configure API Gateway as a SOAP webservice passthrough (does not transform or convert the responses)
- You can increase the throttling limits from the console

# DynamoDB
- Stored on SSD, spread across 3 geographically distinct data centers
- Eventual consistent reads (consistency of all copies is reached within a second usually)
- Strongly consistent reads (read returns a result that reflects all writes that received a successful response prior to reads)
- You can use special IAM Condition to restrict user access to only their own records
    * for instance partition key is user id, you can restrict each user to his own partition
        ForAllValues:StringEquals, dynamodb:LeadingKeys: 

## Indexes
- LSI
    * Local secondary index LSI can only be created when creating the table
    * You cant add remove modify LSIs later
    * Has same partition key as the table, but a different sort key
- GSI
    * can be created anytime
    * choose a new partition key and sort key

## Query
- By default queries return all attributes, use ProjectionExpression to filter the attributes to be returned
- Results are sorted by the sort key (by default in ascending order)
- You need to explicitly set the query to be strongly consistent
- Scan index forward parameter relates to query operation only (to reverse order of results)

## Scan
- Examines all items in the table
- Same as query with ProjectionExpression to select attributes
- Parallel scans, you can configure to use parallel scan by logically dividing table or index into segments and scanning each segment in parallel, if table is busy reading and writing then parallel scan is not suitable as it will affect other applications accessing the table
- By default scan scans the table 1mb at a time
- Reduce the impact of scan/query by using smaller page size

## Provisioned Throughput
- 1 WCU = 1 x 1kb write per second
- 1 RCU = 1 strongly consistent read of 4kb per second
- Eventual consistent reads are default setting
-
    * App needs to read 80 items per second with strong consistency
    * each item is 3kb
    * This means single item needs 0.75 RCU per item, round it up to nearest integer which is 1. This means table needs 80 RCUs.
- For eventual consistent reads, you would need 40 RCUs.
- Same calculation with writes as well, round up to nearest integer, always find the CU needed for a single item first, then round then calculate
- You can alternate between on-demand and provisioned throughput once per day.
- SDK has built-in auto retry for "provisioned throughput exceeded exception"
- All SDKs use exponential backoff in addition to simple retries

## DAX
- Dynamodb accelerator is a fully managed in-memory cache for dynamodb
- Microsecond latency for millions of requests per second
- DAX is write-through cache: it writes to dynamodb and DAX at the same time.
- You can point your app to DAX cluster
- If item is not found in DAX then DAX queries dynamodb with an eventual consistent GetItem
- Using DAX might even reduce the provisioned read capacity
- Not suitable for write heavy apps or requiring strongly consistent reads
- Not suitable for apps that don't need microsecond response time or has very little reads

## TTL

- Define an expiry time and expired items are marked for deletion
- Suitable for session data, event logs, temp data
- You can filter out expired items from queries and scans
- TTL expressed in EPOCH time

## Streams

- DDb streams are time-ordered sequence of item level modifications
- Before and after images are captured
- Logs are encrypted at rest and stored for 24 hours
- Mainly used for triggering events, good for serverless apps
- There's a separate endpoint to access dynamodb streams
- Minimum amount of item you can select to have in the stream is the primary key
- Events are recorded in near real time

# KMS
- Encyrption keys are under IAM but it's not global, it's regional
- Encryption keys for KMS are regional! you can't use eu-west-1 to decrypt data from us-east-1
- Customer master key consists of an alias, creation date, description and key material (customer can provide this or it can be AWS provided). It's created from IAM console
- You can never export customer master key from KMS
- To export you need CloudHSM (hardware security module)
- APIs
    * aws kms encrypt
    * aws kms decrypt
    * aws kms re-encrypt (takes encrypted and first decrypt and reencrypt)
        re-encrypt does not expose the data to the client
        everything happens on server-side
    * aws kms enable-key-rotation
- Envelope encryption
    intermediate key (data key)
    envelope key is used to 
    encrypt the data. master key encrypts the envelope key so you need master key to access the data.
    for decryption: you take master key to decrypt our data key and turn it into plain text and take the plain text data key to decrypt the data
- MASTER KEY CAN'T BE EXPORTED EVER
- You need separate permissions for who can administer the CMK and who can use it to encrypt/decrypt data
- AWS Managed Customer Master Keys cannot be used to directly encrypt and decrypt files, since permissions cannot be changed to them. Customer Managed Customer Master Keys can be used with the KMS command to encrypt the file, when Key Usage rights are given.
- CloudHSM is dedicated hardware, whereas KMS is multi-tenant, they both serve the same purpose

# SQS
- Bisibility timeout: when a consumer picks up the message but does not delete it, it will become visible after this time
- Default visibility timeout is 30 seconds, max can be set as 12 hours
- Messages are up to 256kb
- Standard queues (default, at-least once delivery, best-effort ordering)
- FIFO queues (delivered in the order which they're arrived, order is preserved)
    delivered once, no duplicates
    limited to 300TPS
- Messages can live up to 14 days (retention days)
        * default is 4 days
- Long polling does not return a response until a message arrives or long poll timeouts. This will make it cost efficient to poll SQS
- Long poll max timeout is 20 seconds
- Short polling is default and returns immediately if there are no messages

# SNS
- pub/sub
- Push notifications to devices
- Can deliver notifications via sms, email or sqs or any http entpoint
- Can trigger lambdas
- Topics allow grouping multiple recipients, used for fanout
- Multi-az, messages are stored redundantly
- Instantenous push based delivery (no polling)

# SES
- Incoming mails can be configured to trigger lambdas and SNS notifications
- For email only

# Kinesis
- Think small frequent data from multiple sources
- Retention is by default 24 hours, can be increased up to 7 days
- Producers/consumers

- Kinesis streams, firehose, analytics all three are separate services. Kinesis streams has two options
    * video streams (securely stream video from connected devices to AWS for analytics and machine learning)
    * data streams
- Streams have shards
- A shard gives 5 TPS for reads up to max total data read rate of 2MB per second and up to 1000 TPS for writes up to a max rate of 1mb per second including partition keys
- Capacity of the stream is a function of number of shards
- Firehose is similar, you don't worry about shards, it will store the data in S3 or process by lambda
- To store data in redshift from firehose, you first connect firehose to S3 and then S3 to redshift

- Analytics => S3, redshift or elasticsearch. You use SQL like query.

# Elastic beanstalk
- Elastic beanstalk has 4 deployment options
    * all at once (not ideal, there will be downtime)
        if fails you need to rollback by redeploying original version to all
    * rolling (deployed in batches)
    * rolling with additional batch (maintains full capacity, launches additional batch to do this if you cant tolerate performance degradation while deployment, choose this)
    * immutable (maintains full capacity, preferred for mission critical prod systems, easier to rollback than other options)
- You can customize beanstalk, use the config file in a folder .ebextensions and filename should have name *.config. These are yaml or json. Folder .ebextensions should be on top-level folder
- If you create RDS from within beanstalk env, then it will be deleted when the env is deleted, so it's tied to the lifecycle of the application (suitable for dev/test)
- For prod workflows, create RDS from outside beanstalk to have more flexibility
- To connect RDS and beanstalk when RDS is created from outside the env, you need
    * additional security group added to the auto scaling group
    * provide the connection string information to the env

# SSM - Systems manager parameter store
- Admin needs to store a sensitive parameter etc. such as license keys, passwords etc.
- This is under EC2 console
- You can use this service with EC2, Lambda, CloudFormation etc.

# CI/CD
- Continuous integration: merge code frequently, build and test automated
- Continuous delivery: might include manual approval steps (makes sure changes are ready to go but does not release them)
- Continuous deployment: no manual intervention, everything automated even deployment

## CodeDeploy
- In-place deployment: application is shut down and capacity is reduced, then new version is deployed and started
    * in-place is called rolling as well, to rollback you need to do another deployment. not supported for lambda, only ec2 and on-premises
- Blue/green: new instances are provisioned. blue represents active deployment and green is the new release
    * advantage: release by simply switching all traffic to preprepared servers (new version deployed)
    * faster to rollback (switch back to old version servers)
    * supported for all (including lambda)
- appspec.yml specific the deployment process
- Deployment group: EC2 instances, lambda etc. target for deployment
- Choose EC2 instances by using tag groups
- Deployment settings: oneatatime, halfatatime, allatonce
- appspec: (only YAML for EC2)
```
    hooks:
        beforeallowtraffic: lambdatovalidatebefore
        afterallowtraffic: lambdatovalidateaftertrafficshift
    resources:
        mylambda

    for ec2 and onpremise, appspec should be on the root level folder and following hooks exist:
        beforeblocktraffic, blocktraffic, afterblocktraffic,
        applicationstop, downloadbundle, beforeinstall, install, afterinstall, applicationstart, validateservice,
        beforeallowtraffic,allowtraffic,afterallowtraffic
```

## CodeBuild
- buildspec.yml
```
    phases:
        pre_build:
            commands:
        build:
        post_build
    artifacts
```

## CodePipeline

# CloudFormation
- Resources section is the only mandatory one, all others are optional (Conditions, Parameters, Metadata, Description, Output, AWSTemplateFormatVersion, Mappings, Transform(used to include other templates into current, refer from S3 also to refer to lambda code in S3))
- Nested stack is used with AWS::CloudFormation::Stack resource type
- TemplateURL is mandatory others are not, should point to the stack yml file on S3
- SAM for serverless apps, higher level than CloudFormation, has CLI as well
- sam package: packages app and uploads to S3
- sam deploy: deploys app using CloudFormation CAPABILITY_IAM

# Cognito
- Syncs user data for multiple devices, acts as identity broker between your app and WEB ID providers
- Federation allows users to authenticate with a web id provider, web id provider will receive auth token and then cognito will trade it for temp aws credentials
- Users can sign-in directly to a user pool or indirectly via ID provider
- Successful auth with the pool generates JWTs
- Identity pool is used to receive the temp auth credentials for AWS
- Push synchronization is: cognito uses SNS to send silent notification to the client when data changes

# IAM
- 3 type of policies: managed, customer managed, inline
- AWS provides managed policies based on job function such as AdministratorAccess
- You can't change permissions defined in a managed policy
- Inline policies are specific to a user, you cant attach it to multiple users and when the entity (user) is deleted the policy will also be deleted

## STS AssumeRoleWithWebIdentity API
- Api provided by sts, returns temp security credentials for users authenticated by a mobile or web app or using a web id provider
- If you use cognito you dont need to explicitly use this API, its used behind the scenes
- App authenticates with Facebook and receives JWT, then passes JWT to this api to receive session token to be used with AWS as temporary credentials
- This API response includes AssumedRoleUser ARN and AssumedRoleID to be used programmatically to reference the temporary credentials. These are not IAM role or user!

## Cross account access (Say Dev and Prod)
- To set this up you should identify the account numbers, create group in IAM (dev), create user in IAM (dev), log in to Prod account and create the policy. Then create UpdateApp cross account role and apply the policy to the role in Prod account. Then login to developer account and create new inline policy and apply to dev group, login as john and switch accounts

# CloudWatch
- RAM utilization is a custom metric. (can see CPU, network and overall disk throughput)
- By default EC2 monitoring is 5 minute, if you enable detailed monitoring you get 1 minute intervals
- By default log data retention is indefinite (never deleted)
- You can retrieve data from any ec2 or elb instances after its termination
- CloudWatch can be used on premise as well, you need to install the SSM agent and CloudWatch agent.

# Config
- AWS Config records the state of your AWS environment and can notify you of changes

# Others

## SQS
- SQS delay queues are queues where new messages to the queue is delayed for a period of time, default delay is 0 seconds and max is 900 seconds
- For FIFO queues this affects the delay of messages already in the queue, for standard queues it does not affect messages already in the queue
- For large messages, use S3 to store the messages and use Amazon SQS Extended Client Library for Java to manage them
- This can handle 256KB up to 2GB in S3.
- You'll also need the AWS SDK for Java it provides an API for S3 bucket and object operations
- Using the Extended library you can specify a threshold for which messages to be stored on S3 or you can do all in or none

## CLI Pagination
- By default returns 1000 items per API call (called the page size)
- If you get timeout errors or too many results errors, use --page-size to reduce the page size
- The CLI still retrieves the full list but performs a larger number of API calls

## IAM Policy simulator
- Test IAM permissions before you commit them to prod
- Validate policy works as expected
- You can also test policies attached to existing users, great for troubleshooting

## Kinesis Shards & Consumers
- Kinesis Client Library runs on consumer instances
- Tracks number of shards
- Discovers new shards when you reshard
- KCL ensures that for every shard there's a record processor
- Manages number of processors relative to # of shards & consumers
- If you have only one consumer, KCL will create all record processors on a single consumer
- If you have two consumers, it will load balance and create half of processors on one instance half on other
- With KCL, you generally need to ensure # of instances does not exceed # of shards (except for failover or standby purposes)
- Never need multiple instances to process load of a single shard
- One worker can process multiple shards however
- CPU utilisation is what should drive quantity of consumer instances, not the # of shards!
- Use autoscaling group and base the decision on CPU load on consumers

## Lambda concurrent executions limits
- This is a limit across all functions in a given region per account
- Default is 1000, if you hit the limit you'll get TooManyRequestsException and HTTP429
- You can request limit to be increased
- You can reserve concurrency for a critical function to make sure it always gets the  # of executions it needs, however this also acts as a limit. So if you set 600 for a function, it wont be able to do 601

## Lambda and VPCs
- Imagine your lambda needs to access resources in a VPC
- Lambda needs following VPC config information to connect to the VPC:
    * Private subnet ID
    * security group ID (with required access)
    * Lambda will use this info to set up ENIs using an available IP address from your private subnet
    * you can use the aws lambda update-function-configuration --function-name yourfunc --vpc-config SubnetIds=dsfsfsd,SecurityGroupIds=dfasfs
- Your lambda will need ec2:CreateNetworkInterface, ec2:DeleteNetworkInterface and ec2:DescribeNetworkInterfaces

## XRAY
- For using x-ray from ECS, you should create a new docker container for the daemon on the ECS cluster
- You can add additional data via annotations abour requests (like game name, game id etc.) then you can filter using these annotations
- These are in the form of key-value pairs

## Deploying Docker using Elastic Beanstalk
- You can deploy multiple docker instances to your ECS cluster
- Using Elastic Beanstalk, you can deploy your docker container to a single EC2 instance