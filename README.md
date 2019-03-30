# AWS Developer: Getting Started

> My notes from this [Pluralsight course](https://app.pluralsight.com/library/courses/aws-developer-getting-started/table-of-contents)

## Welcome to AWS

### Developing in the Cloud

**Similar**

- Application runs on OS/Platform
- Requires platform to be installed (eg: .NET)
- HTTP servers listens on ports
- Connects to outside databases

**Different**

- Locally everything runs on same machine, AWS one service === one responsibility
- Local files stored on server, AWS files served from S3
- Local db on same machine as app, AWS database as a service
- Local connections done manually, AWS connections doen with AWS SDK

- AWS Access Key gives access for SDK & CLI (will use js SDK for this course)

### Sample App

Start with sample node.js app that runs locally, then will be extending it to work in the cloud.

- will use EC2 for app hosting
- images and assets will use S3
- user info will be saved in RDS
- user login session will be stored in ElastiCache to avoid wasting app memory
- topping selection (pizza app) will be saved in DynamoDB

![sample app architecture](doc-images/sample-app-architecture.png 'sample app architecture')

### Creating and Initializing AWS Account

[Install AWS CLI](https://aws.amazon.com/cli/)

```shell
$ python --version
Python 3.6.4
$ pip install awscli
$ aws --version
aws-cli/1.16.121 Python/3.6.4 Darwin/16.7.0 botocore/1.12.111
```

- Navigate to [https://console.aws.amazon.com](https://console.aws.amazon.com), create account.
- Generate AWS access key and save it somewhere secure (will not be able to view secret again)
- Use cli to configure, providing access key, secret, region (eg: `us-east-1`) and default output format (eg: `json`)

```shell
$ aws configure # follow prompts
$ aws ec2 describe-instances # don't have any yet, just to test its configured properly
{
    "Reservations": []
}
```

## Sounding the Alarm with IAM and Cloudwatch

### CloudWatch Overview

- Service to set alarms for various metric thresholds, eg: CPU Usage on EC2 instances, Provisioned throughput on DynamoDB tables, Billing charges etc.
- Use Simple Notification Service (SNS) to configure where notification should be sent to
- \$0.10/alarm/month

![cloud watch](doc-images/cloudwatch.png 'cloud watch')

Can configure alarm to trigger an action, eg, on increased network traffic, auto scale

![cloud watch auto scale](doc-images/cloudwatch-auto-scale.png 'cloud watch auto scale')

### Simple Notification Service

- Companion to CloudWatch, push alerts to a group of notification endpoints
- _Topic_ - used to create a unique amazon resource name that can have notifications sent to it
- Subscribe to topic via email or SMS, then any notification sent to topic will be communicated to subscribed endpoints
- Topics are region specific, can only be notified by alarms in same region

![sns](doc-images/sns.png 'sns')

**Demo: Create billing alarm (only available in N Virginia region)**

Web console has changed a lot since course was published.

- Services menu -> SNS
- Create topic, eg: `admin_email`
- Create subscription, Protocol: email
- Enter email adress, then go to that address and confirm subscription

### Creating a CloudWatch Alarm

- Setup a billing alarm
- Services -> CloudWatch
- Billing alerts need to be enabled from billing dashboard in your account
- Your name -> Billing
- Preferences -> Receive Billing Alerts
- Back to CloudWatch -> Alarms -> Create Alarm
- Only shows you metrics for resources you've created (eg: Billing Metrics)
- Total Estiated Charge -> USD
- Set time range 12h
- Name: Billing Alarm
- Threshold, eg: >= \$5
- No longer have to select Topic from SNS
- When threshold has passed, alarm will go into `ALARM` state and notification will be sent to the selected Topic

### Identity & Access Management Overview

- IAM: Access management service, controls who can have acess to what resources in AWS account
- Gatekeeper to all AWS services
- Manage access to users on your account - passwords, MFA, access keys, ssh keys
- _Policy_ - collection of permissions to access different services in a particular way
- Policy can be specific to resource level, or braod to allow permissions on a type of service, usually mixed and matched to a particular group
- eg: Developer Group Policy: Full access to EC2 to spin up instances as needed
- eg: Tester Group Policy: Read only access to DynamoDB, full access to some S3 bucket
- Access from console: Security -> IAM

### Securing Your AWS Account

- Root Account Permissions - base of account, this is what you get when you first create an AWS account, very powerful account - administer all services, modify billing info, manage users
- Setup MFA (Multi-factor authentication) for root account

### Understanding Policies

- Building block of IAM permissions
- One or more statements that declare whether user is allowed to do something or not
- Default: Users have no permissions
- Policies can be applied to individual users or groups (which users can be added to)
- Use policies to give permissions
-

**Policy Statement Properties**

1. Effect: Allow or Deny (denied by default)
2. Action: Operation user can perform on services, each service has set list of actions that can be performed on it
3. Resource: Specific resource user can request to perform action on, resource property should have ARN entered (Amazon Resource Name). Can set to `*` to give user access to any resource.

**IAM Policy Types**

AWS comes with some pre-created policies, but if you want to restrict user to a specific resource, need to define custom.

- AWS Managed Policy: General purpose, service-wide permissions
- Customer Managed Policy: Specific resource permissions

**Demo**

- Dashboard: Security -> IAM -> Left side Policies
- View `AdministratorAccess` policy
- Policy Document - JSON, `Statement` is array of access objects, eg:

```javascript
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        }
    ]
}
```

- Filter by EC2 -> select AmazonEC2FullAccess (more specific example)

```javascript
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "ec2:*",
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "elasticloadbalancing:*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "cloudwatch:*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "autoscaling:*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:CreateServiceLinkedRole",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": [
                        "autoscaling.amazonaws.com",
                        "ec2scheduled.amazonaws.com",
                        "elasticloadbalancing.amazonaws.com",
                        "spot.amazonaws.com",
                        "spotfleet.amazonaws.com",
                        "transitgateway.amazonaws.com"
                    ]
                }
            }
        }
    ]
}
```

### Configuring Users and Groups

- Manage authentication and access for a single entity (could be person or resource)
- Apply policies to groups of users to reduce complexity
- Individual users can be added to more than one group
- Best practice: DO NOT USE YOUR ROOT ACCOUNT TO DO THINGS IN AWS, because it has unlimited permissions to do anything
- Recommended: Create a new user login under your root account
- Console -> Security -> IAM: Select Users from left hand side, create new user
- Forced to also create a group, add EC2 policy to group, then add user to this group
- Generate access key for each user (will be replacing the key we created earlier for CLI and SDK)
- Run `aws configure` again, replacing key values with those of newly created user
- Try test `aws ec2 describe-instances`, should output same result because user has EC2 policy
- Go back to console, delete root access key: User name -> Security...
- IAM dashboard -> left side -> Account Settings -> Password Policy -> customize to whatever you need

For remainder of course, will be move convenient for user to have AdministratorAccess policy, later can customize for "least privilege". Then login as that user for rest of course.

Note that `https://console.aws.amazon.com` is only for root user login. For regular users, use url shown at top of IAM dashboard, eg: `https://123456.signin.aws.amazon.com/console`

## Getting Inside the Virtual Machine with EC2 and VPC

### Virtual Private Cloud Overview

- Create virtual network where resources can be launched, and be isolated from other VPC's and outside world
- Defines private logical area for instances
- Within VPC, resources have private IP addresses to communicate with eachother
- Can control to all resources inside VPC
- Can route outgoing traffic as desired
- VPC is free service on AWS, max of 5 VPC's per account

![vpc isolation](doc-images/vpc-isolation.png 'vpc isolation')

![vpc ip](doc-images/vpc-ip.png 'vpc ip')

**Security Group**

- Key structure to configure VPC: Defines collection of IP's that are allowed to connect to your instance, and IP's that instance is allowed to connect to (mini-firewall)
- Security groups are attached at instance level, can be shared among instances
- Can set security group rules to allow access from other security groups instead of by IP

**Routing Table**

- Used to configure routing destinations for traffic coming out of VPC
- Each VPC has one routing table - declares attempted destination IP's and where they should be routed to
- Eg: run all outgoing table through proxy

**Network Access Control List**

- Another tool used by VPC
- IP filtering table - specifies which IPs are allowed through VPC for incoming and outgoing traffic
- "super powered security group" - apply rules for entire VPC

**Subnet**

- Instances can't be launched into a VPC, but rather, a subnet that is inside VPC
- Additional isolated area with its own routing table and network access control list
- Use to create different behaviour within same VPC

![vpc subnet](doc-images/vpc-subnet.png 'vpc subnet')

![vpc example](doc-images/vpc-example.png 'vpc example')

### Creating a Virtual Private Cloud

For course, will create simple VPC with 2 public subnets

![sample architecture](doc-images/sample-architecture.png 'sample architecture')

Web Console -> Services -> VPC

Resource Summary shows default VPC but will create one from scratch.

- Click Launch VPC Wizard button
- Select VPC with a Singe Public Subnet
- IP CIDR block - Range of Available IP Addresses
- VPC name: pizza-vpc
- CIDR block of subnet is a subset of VPC CIDR block
- Select first availability zone
- Subnet name: pizza-subnet-a
- Click Create VPC button
- Need to add an entry to routing table to allow all traffic to go through internet gateway, to public internet
- Click "Your VPCs" menu option from left, select pizza-vpc -> main route table -> Routes tab
- By default, there's a single entry to specify that any IP address referencing local vpc CIDR block should resolve locally
- Add new entry
- Destination: 0.0.0.0/0 (anywhere)
- Target: igw... (provided internet gateway)

Now VPC is configured to let outgoing traffic get out to internet.

- Will be launching app using auto scaling groups -> must deploy into multiple availability zones
- A subnet can only exist in a single availability zone
- Therefore, need to create more subnets to have ability to deploy app into multiple availability zones

Crate Public Subnet for Scaling

- Name tag: pizza-subnet-b
- VPC: pizza-vpc
- Availbility Zone: Select a different entry than previous
- CIDR block: Important to avoid IP conflict between subnets, 10.0.1.0/24 -> provides 255 more IP addresses that are different than previous subnet

### Elastic Cloud Compute (EC2)

- Just a virtual machine, provisioned with some amount of CPU cores, memory, storage space and network performance
- Can put whatever OS you want, usually Linux (Amazon, Red Hat, Ubuntu etc) or Windows
- Licensed OS will cost more per hour, Amazon's flavour of Linux is cheapest
- AMI: Amazon Machine Image - OS + software installed on EC2 instance
- EBS: Elastic Block Store: EC2 instances need some storage for OS and app code to be uploaded, this storage lives independently of the instance, so could live on even if instance is terminated

**EC2 Dashboard**

- Shows number of running instances, volumes
- Launch Instance button to launch a new instance

### Creating an EC2 Instance

- First, setup EC2 as favourite, click Pin icon, then drag EC2 to menu bar
- Click Launch Instance button to start launch wizard
- Select Amazon Linux image
- Select `t2.micro instance` (only one available for free tier)
- Number of instances: 1
- Network: pizza-vpc
- Subet: pizza-subnet-a
- Auto-assign Public IP: Disable
- Leave defaults for remainder of options
- Accept default for Storage (8GB)
- Add name tag: pizza-og
- Security Group name: pizza-ec2-sg
- Security Group description: security group for pizza luvrs ec2 instances
- Rules: Leave the default SSH rule (for production would restrict which IPs could SSH), then add rule to open port
- Type: Custom TCP Rule, Port Range: 3000, Source: Anywhere
- Click: Review and Launch, then click Launch
- Create key pair for instance, name: pizza-keys, then download key pair -> pem file

### Connecting to an EC2 Instance

First download exercise files to get sample pizza project from pluralsight or [github](https://github.com/ryanmurakami/pizza-luvrs), then:

```shell
cd pizza-luvrs
npm install
npm start
# open browser at http://localhost:3000/
```

- Back in AWS Console, go to EC2 dashboard -> should be one running instance, click to select it
- Note there is no public IP, but there is private IP in range that was configured for VPC/subnet
- To ssh into it, first need to create and assign a public IP address

**Elastic IP**

- Public IP addresses that can be associated with instance
- If underlying instance is terminated, IP still exists and can be associated with a different instance
- Select Elastic IP from left menu of EC2 dashboard (under Network & Security)
- Click Allocate new address
- Once allocated, click Actions -> Associate address
- Instace: pizza-og
- Copy IP address

**Connect to the EC2 Instance via SSH**

```shell
chmod 400 ~/pem/pizza-keys.pem # change permissions of pem file so only readable by user
ssh -i ~/pem/pizza-keys.pm ec2-user@replace.with.elastic.ip
```

### Updating and Deployingto an EC2 Instance

- First need to update OS software and install Node.js (course using older 6.x), from ssh session:

```shell
$ sudo yum update
$ curl --location https://rpm.nodesource.com/setup_6.x | sudo bash -
$ sudo yum install -y nodejs
```

**Transfer Demo Application Code to EC2 Instance**

`exit` to end ssh session and return to local system prompt

```shell
cd pizza-luvrs
rm -rf node_modules
cd ..
scp -r -i ~/pem/pizza-keys.pem ./pizza-luvrs ec2-users@replace.with.elastic.ip:/home/ec2-user/pizza-luvrs
ssh -i ~/pem/pizza-keys.pem ec2-user@replace.with.elastic.ip
cd pizza-luvrs
npm install
npm start
# Open browser at http://replace.with.elastic.ip:3000
```

### Scaling EC2 Instances

- Don't want to rely on a single instance to run app because something could go wrong with it
- Would like image installed on instance to be OS + required software such as Node + app, so the whole thing is ready to go
- Can do this by creating an Amazon machine image from an existing EC2 instance: Custom AMI -> EC2 instance saved as snapshot and replicated
- However, don't want to do this manually...

**Auto Scaling Group**

- Expand or shrink a pool of instances based on pre-defined rules
- Uses launch configuration which has an image, and scaling rules which will take care of starting/stopping instances automatically
- But how to tell users where app is, since IP's could keep changing as result of scaling...

**Load Balancer**

- Routing appliance that maintins consistent DNS entry, and balances requests to multiple instances
- Keeps track of which IPs are available and redirect user traffic to those
- Connects to auto scaling group, LB and auto scaling group work together to create groups of instances and route users to them

### Creating an Amazon Machine Image (AMI)

- Create an AMI from an existing EC2 image
- EC2 dashboard -> Instances
- Select `pizza-og` instance created earlier
- Actions dropdown -> Image -> Create Image
- Image name: pizza-image
- Leave other defaults like volume
- Click Create
- Now if click Launch Instance -> My AMIs, will see custom AMI: `pizza-image`
- Can also see it by clicking on AMIs section of left menu (from EC2 dashboard)

### Creating a Load Balancer

- EC2 Dashboard -> Click Load Balancers from left menu -> Create Load Balancer
- Load Balancer name: pizza-loader
- Create LB Inside: select pizza-vpc
- Create an internal load balancer: leave UNCHECKED, want this LB open to public
- Listener Configuration: Which ports LB should listen on? (usually HTTP on port 80 and HTTPS on port 443)
- For this course don't have SSL cert so will just use HTTP/80
- For each listener: Define protocol/port LB should listen for, THEN what protocol/port LB should send to the instances.
- Since sample app listens on 3000, LB listener: HTTP 80 HTTP 3000
- Select Subnets LB should send traffic to: the two subnets created earlier in course
- Click Next
- Create new security group for LB
- Security group name: pizza-lb-sg
- Leave default rule: Custom TCP Rule, Protocol TCP, Port Range 80, Source: Anywhere
- Click Next, again Next
- Configure Health Check - tell LB which path to hit to verify instances are alive and well
- Ping Protocol HTTP, Ping Port 3000, Ping Path /
- Leave Advanced Details as per defaults
- Add EC2 instances: Leave unchanged because want to use this LB with auto scaling group (will create in next section)
- Don't need tags
- Click Review & Create, then Create

**Enable Instance Stickiness on Load Balancer**

- Ensure that once user hits one EC2 instance via LB, they should continue to hit that same one on subsequent requests. (Needed because simple demo app used for course doesn't use a persistent session store like Redis)
- LB will use cookie to keep track of which user should be sent to which EC2 instance
- EC2 Console -> Load Balancers -> select LB just created
- Details panel: Port Configuration: Stickiness Diabled, click Edit
- Select option: Enable load balancer generated cookie stickiness
- Expiration Period: 86400 (seconds) this value matches session expiry of 1 day within app
- Click Save

### Creating an Auto-scaling Group

- to use with LB
- EC2 Console -> Auto Scaling Groups from left side menu
- Click Create Auto Scaling Group
- This wizard will be used to create both launch configuration and auto scaling group
- Click Create launch configuration (defines how each instance will be created from auto scaling group)
- Select My AMIs from left -> pizza-image
- Select t2.micro
- Name: pizza-launcher
- Advanced Details: Very important, not enough just to create instance, but also need to start node app
- User data: Run scripts to execute at instance startup:
  ```shell
  #!/bin/bash
  echo "starting pizza-luvrs"
  cd /home/ec2-user/pizza-luvrs
  npm start
  ```
- Click Next
- Leave storage volume options as is, click Next
- Select an existing security group: pizza-ec2-sg
- Click Review
- Click Create launch configuration
- Select pizza-keys key pair to start each instance with
- Check acknowledgement
- Create launch configuration
- NOW we can Create Auto Scaling Group
- Group name: pizza-scaler
- Group size: Start with 2 instances
- Network: pizza-vpc
- Subnet: Click in input field, then select pizza-subnet-a, then pizza-subnet-b
- Ignore warning about internal instances not having public IP addresses, this is fine, only want LB to have public IP address
- Expand Advanced Details
- Check "Receive traffic from Elastic Load Balancers"
- Click in input and select: pizza-loader
- Leave remaining options as default and click Next
- Click Next again (will configure scaling rules later)
- Do not configure notification on scaling up/down for now but could do later if you want, click Next
- Click Review, Create Auto Scaling Group

Looking at list of Auto Scaling Groups, at fisrt will show:
Instances: 0
Desired: 2
Min: 2
Max: 2

Refresh after some time, now instances should show 2

Also check on Load Balancer (from left menu) -> should show 2 of 2 instances in service

Load Balancer Description tab shows DNS name, eg: `pizza-loader-someid.us-east-2.elb.amazonaws.com (A Record)`, use this to access app via browser. For a real app, would have a domain like pizzaluvrs.com and configure CNAME.

**Adjust Security Groups**

- From EC2 Console - Security Groups from left menu
- Click on `pizza-ec2-sg`
- Select Inbound tab
- Leave SSH rule as is
- Change Custom TCP Rule (port 3000) to only accept connections from LB
- Change Source to Custom, Input: type `s` to bring up auto suggest security groups,
- Select security group from load balancer -> i.e. any instance having LB security group will be allowed to connect to this EC2 instance on port 3000. This gives access without being tied to an IP address

### Scaling in Action

- Configure scaling rules for auto scaling group, currently set to static `2` instances.
- From EC2 Dashboard, click Auto Scaling Groups left menu option and select `pizza-scaler` auto scaling group.
- Details Panel -> Scaling Policies tab -> click Add policy
- For this course, will use "Create a simple scaling policy" <- Click this link, notice some advanced options are removed
- Name: scale up
- Execute policy when: (select CloudWatch alarm that will trigger scaling action, but we haven't yet created), click "Create New Alarm"
- Uncheck "Send a notification to" (for a real app, would want this)
- Set which metric to look at for instance, options include:
  - CPU Utilization <- Also ok
  - Disk Reads
  - Disk Read Operations
  - Disk Writes
  - Disk Write Operations
  - Network In
  - Network Out <- Good judge of how hard instance is working
- Whenever: Average
- of: Network Out
- Is: >=
- 1000000 Bytes (not realisitic, but for this course, allows us to see scaling in action)
- Leave period setting as is: For at least 1 consecutive period of 5 minutes
- Name of alarm: (leave as is): awsec2-pizza-scaler-High-Network-Out
- Click Create Alarm
- Back on Create Scaling policy dialog...
- Take the action: Add
- 1 instances
- Click Create
- Add one more scaling policy to scale back down...
- Click Add policy
- Name: scale down
- Click Create new alarm
- Uncheck notifications
- Whenever: Average
- of: Network Out
- Is: <
- 1000000 Bytes
- For at least: 1 consecutive period(s) of: 5 Minutes
- Name of alarm: awsec2-pizza-scaler-Low-Network-Out
- Click Create Alarm
- Again on Create Scaling policy dialog...
- Click Create a simple scaling policy link
- Take the action: Remove 1 instances
- Click Create
- Need to adjust auto scaling group min/max instances to allow scaling policy to take effect...
- Make sure `pizza-scaler` is selected, then click Actions -> Edit
- Change Max to 4 instances
- Click Save

**Generate Requests to the Application**

To see the scaling in action, need to simulate load on the app, can use automated tools such as:

- JMeter
- Apache Benchmark

For this course, will use Apache Benchmark. Send 8000 requests to LB at a max of 5 concurrent at a time. This will trip network alarm of >= 5M bytes:

```shell
ab -n 8000 -c 5 http://pizza-loader-replace-with-id-and-region.elb.amazonaws.com/
```

Watching EC2 console on Auto Scaling Groups, takes ~5 min for CloudWatch alarm to do aggregating before triggering. Refresh activity after few minutes to see new instance launched, then refresh again few minutes later to see new instance terminated.

## Hosting All the Things with S3

### S3 Overview

- Simple Storage Service
- Stores objects
- Easily configure who can see objects
- Stores objects in specified regions
- Object: Fundamental S3 storage structure
- Object === File + Metadata
- Metadata includes: File type, modified date, custom
- File: Can be anything like jpg, txt, json, js
- Max file size is 5TB
- PUT limit 5GB
- Objects stored in _Bucket_
- Bucket: Fundamental S3 storage structure
- Buckets contain multiple objects
- Buckets also create url namespace for objects
- Eg: Region = Oregon, Bucket Name: pizza-luvrs, Root URL: `s3-us-west-2.amazonaws.com/pizza-luvrs`
- Bucket name must be unique throughout ALL of S3, not just in your account
- Objects uniquely identified via key: path + filename (excluding bucket name)
- Eg: Filename = image.png, Folder name: images, Object Key = images/image.png

**Permissions**

- Default - Buckets and Objects only viewable by AWS account that created them
- With Permissions, can grant access to another AWS account or anyone (Anonymous)
- Usually for web resources, want viewable by everyone, but not modifiable
- Can also use IAM policy, eg: `AmazonS3FullAccessPolicy`

**Latency**

- If object is stored for example in Oregon, USA region, user requesting it from Sydney, Australia will experience high latency
- Can solve with cross region replication - configured feaature of S3, automatically copy files added/modified in S3 bucket to a different region. BUT will only copy to one other region
- CloudFront is best way to solve geographic latency (discussed later in course)

### Creating an S3 Bucket

- click Create Bucket
- Bucket Name: pizza-luvrs-something-unique
- Region: Ohio
- Don't check off logging
- click Create
- Edit Bucket Permissions (access to view bucket's contents and make changes)
- Each Object in bucket also has permissions - more fine grained control
- Allow bucket to be completely bucket
- Apply single permissions policy to give anyone access to view every object in this bucket
- AWS Policy Generator: [https://awspolicygen.s3.amazonaws.com/policygen.html](https://awspolicygen.s3.amazonaws.com/policygen.html)
- Policies are json but tediouis to create by hand, use policy generator instead...
- Select Type of Policy: S3 Bucket Policy
- Effect: Allow
- Principal: \*
- AWS Service: Amazon S3
- Action: GetObject
- ARN: arn:aws:s3:::pizza-luvrs-something-unique/_ (/_ is important -> apply to any object in this bucket)
- Click: Add Statement
- Click: Generate Policy
- Copy JSON
- Back in S3 dashboard
- Click Add Bucket Policy
- Paste in JSON from generator
- Click Save

### Uploading Objects to S3

Need to move various resources used by pizza-luvrs app to S3, then change all references in code to use S3 url's, eg:

- `/pizzas/*.png`
- `/toppings/*.png`
- `/header_logo.png`
- `css/stylesheet.css`
- `js/make.js`

Upload can be via AWS console (good for adhoc, small number of files), CLI (better for bulk uploading) or SDK.

For CLI, general form of upload command is: `aws s3 cp <local_folder> s3://<bucket>/<remote_folder> --recursive --exclude "<pattern>"`

From pizza-luvrs project root, will upload js, css and pizza assets:

```shell
$ aws s3 cp ./assets/js s3://pizza-luvrs-something-unique/js --recursive --exclude ".DS_Store"
upload: assets/js/make.js to s3://pizza-luvrs-something-unique/js/make.js
$ aws s3 cp ./assets/css s3://pizza-luvrs-something-unique/css --recursive --exclude ".DS_Store"
upload: assets/css/stylesheet.css to s3://pizza-luvrs-something-unique/css/stylesheet.css
$ aws s3 cp ./assets/pizzas s3://pizza-luvrs-something-unique/pizzas --recursive --exclude ".DS_Store"
upload: assets/pizzas/filthy-rich.png to s3://pizza-luvrs-something-unique/pizzas/filthy-rich.png
upload: assets/pizzas/best-pizza.png to s3://pizza-luvrs-something-unique/pizzas/best-pizza.png
...
$ aws s3 cp ./assets/toppings s3://pizza-luvrs-something-unique/toppings --recursive --exclude ".DS_Store"
upload: assets/toppings/green_pepper.png to s3://pizza-luvrs-something-unique/toppings/green_pepper.png
upload: assets/toppings/banana_pepper.png to s3://pizza-luvrs-something-unique/toppings/banana_pepper.png
...
```

Use AWS console to manually upload `assets/header.png` and `assets/logo-header-png`.

**PERMISSION NOTE**

AWS CLI has the permissions given to the user whose key is configured on your system.

Now S3 bucket in console shows `js`, `css` and `pizzas` folders. Click on any folder to see details, click on file to get Properties, then click on URL, should load in new tab.

### Connecting to S3 with Code

- Modify code to use assets from S3 rather than locally. Need to define the root path, this will be common throughout the code.
- Select any file in S3 bucket from console -> Properties -> copy Object URL up to file, eg: https://s3.us-east-2.amazonaws.com/pizza-luvrs-something-unique. This is _S3 Root Path_.

Modify all files that reference asset paths:

- layout.hbs
- pizza-make.hbs (two refs)
- index.hbs
- partials/header.hbs
- assets/js/make.js -> client side js, must be re-uploaded to S3 after making changes
- lib/imageStoreFile.js -> local implementation, create in same dir `imageStoreS3.js`
- (new) imageStoreS3.js will use `aws-sdk` to save generated pizza images in S3
- modify imageStore.js to use imageStoreS3 instead of imageStoreFile.js

### Working with CORS in S3

Cross origin resource sharing - concern when requesting resources from another domain.

Trying to access S3 images will fail if CORS disabled because S3 is on a different domain than pizza-luvrs app.

![cors disabled](doc-images/cors-disabled.png 'cors disabled')

![cors enabled](doc-images/cors-enabled.png 'cors enabled')

To enable CORS:

- AWS console -> S3 -> select bucket -> Permissions
- CORS configuration -> enter the following:

```xml
<CORSConfiguration>
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>GET</AllowedMethod>
        <MaxAgeSeconds>3000</MaxAgeSeconds>
        <AllowedHeader>Authorization</AllowedHeader>
    </CORSRule>
</CORSConfiguration>
```

### Accessing S3 with EC2

- Have made code changes to app to work with S3, now need to re-deploy app to EC2.
- Complication: When running locally, uses local credentials, but when running on EC2, doesn't have these credentials
- Need to use IAM Role -> Assign IAM Role at launch configuration level (newly launched instances)

STEPS:

1. Update pizza-og EC2 instance with new mode
2. Create new custom AMI from updated instance
3. Create new launch configuration to attach to pizza auto scaling group

Copy updated pizza-luvrs app to EC2:

```shell
rm -rf node_modules
scp -r -i ~/pem/pizza-keys.pem ./pizza-luvrs ec2-users@replace.with.elastic.ip:/home/ec2-user/pizza-luvrs
ssh -i ~/pem/pizza-keys.pem ec2-user@replace.with.elastic.ip
cd pizza-luvrs
npm install
```

Now in AWS Console:

- EC2 -> Instances -> Select `pizza-og` instancee
- Actions -> Image -> Create Image
- Image name: pizza-plus-s3, Click Create Image
- Services -> IAM
- Select Roles from left (Role like a user but doesn't have a way to login, used to attach policies to)
- Create Role
- Role Name: pizza-ec2-role
- Next
- Select Amazon EC2
- Attach policy: AmazonS3FullAccess
- Next -> Create Role

Back in EC2 Console, create new launch configuration to create instances with new AMI and role created in previous steps.

- From left menu, click Launch Configurations
- Create Launch Configuration
- Select My AMIs -> pizza-plus-s3
- Leave t2.micro selected
- Name: pizza-launcher-2
- IAM role: pizza-ec2-role
- Advanced:

```bash
#!/bin/bash
echo "starting pizza-luvrs"
cd /home/ec2-user/pizza-luvrs
npm start
```

- Instances launched by this launch configuration need to go to public internet for S3 access. So must assign public IP address: Select radio button: Assign a public IP address to every instance
- Next: Default storage
- Select an existing security group -> pizza-ec2-sg
- Create

Replace launch configuration in auto scaling group created previously with new one and refresh instances:

- Still in EC2 console -> Auto Scaling Groups
- Select `pizza-scaler`
- Details tab: Edit: Launch Configuration: pizza-launcher-2 -> Save
- Terminate existing Instances

## A Tale of Two Databases with DynamoDB and RDS

### Relational Database Services (RDS) Overview

- DB health - software updates, performance tuning, backups.
- Lots of extra work for a team.
- Amazon solution: Relational Database Service (RDS): Managed database instances in AWS running on EC2
- RDS Managed Task Examples: Software upgrades, nightly db backups, system health monitoring

**RDS Instance Architecture**

- EC2 instance (must choose size, eg `r3.large`) with OS an DB engine.
- Size matters - underpowered EC2 instance won't be able to process queries well -> impact performance

![RDS Architecture](doc-images/rds-architecture.png 'RDS Architecture')

**RDS Backups**

- By default, RDS backups occur daily
- Configurable backup window
- Bakcups stored 1 - 35 days
- Restore db from backup

**Multi-AZ Deployment**

- Option to turn on
- DB replication to different availablity zone (AZ) in same region
- Automatic failover in case of catastrophic event

**Database Read Replica**

- Non-production copy of database
- Eventual consistency with source
- Useful for running queries (eg: analytics) on data without impacting performance of main operational DB
- NOT used as failover

### Choosing a Database Engine in RDS
