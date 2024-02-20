 
Introduction
When building a cloud-based application, the underlying architecture and environment are just as critical as the application itself. There are many considerations when deciding on the proper architecture of your app:
Scalability: How easily and/or frequently does the app need to scale up or down? How much value do you put into not having to constantly micro-manage and monitor resource usage?
Availability: How readily available is your app? How important is being able to go through long periods of time without failures? If failure does occur in a part of your app, how vulnerable is the rest?
Security: How secure is your app? How does your app handle security permissions for different parts of your app? If an attack happens in one part of your app, how vulnerable is the rest?
Enter the AWS 3-tier architecture
Why 3-tier? This form of architecture addresses all the issues stated above. it provides increased scalability, availability, and security by spreading the application into multiple Availability Zones and separating it into three layers that serve different functions, independent of each other. If an AZ does down for some reason, the application has the ability to automatically scale resources to another AZ, without affecting the rest of the application tiers. Each tier has its own security group that only allows the inbound/outbound traffic needed to perform specific tasks.
Web/Presentation Tier: Houses the user-facing elements of the application, such as web servers and the interface/frontend.
Application Tier: Houses the backend and application source code needed to process data and run functions.
Data Tier: Houses and manages the application data. Often where the databases are stored.
The scenario
So what does this mean for you? You’re a Dev Ops Engineer for a new, rapidly growing tech startup. The Product team wants to build a new web app. You’re tasked with planning and building the architecture the application will be run on.
Let's get started!
Prerequisites
An AWS account with IAM user access.
Familiarity with the AWS Management Console.
Familiarity with VPC network structures, EC2 instances, Auto scaling groups, and security groups.
Familiarity with Linux commands, scripting, and SSH.
Access to a command line tool.
The underlying network architecture
Like any overused cliché, you can’t build a house without a solid foundation. To make things a bit easier down the road, we’re going to create the base environment upon which our 3-tier application architecture will be built.
This base network consists of:
A VPC.
Two (2) public subnets spread across two availability zones (Web Tier).
Two (2) private subnets spread across two availability zones (Application Tier).
Two (2) private subnets spread across two availability zones (Database Tier).
One (1) public route table that connects the public subnets to an internet gateway.
One (1) private route table that will connect the Application Tier private subnets and a NAT gateway.

 




 

 
Enable auto-assign IPv4
Once all the assets have been created, to make sure ‘Enable auto-assign public IPv4 address’ for BOTH public subnets so we can access its resources via the Internet.

 
Set main route table
When a VPC is created, it comes with a default route table as its ‘main table.’ However, we want our public-rtb to serve as the main table, so select the public-rtb from the ‘Route tables’ dashboard and set it as the main table under the ‘Actions’ dropdown menu.

 
Create a NAT Gateway
A NAT gateway allows instances from the private subnets to connect to resources outside of the VPC and the Internet (for necessary services such as patches or package updates).
It’s best practice to maintain high availability and deploy two NAT gateways in our public subnets (one in each AZ); however, for now, we will just deploy one.
Navigate to ‘NAT Gateways’ and create a new gateway called pubNAT1. Select one of the public subnets, allocate an elastic IP, and create the gateway.

 

Configure private route tables
As we can see, a route table has been created for each private subnet (4) by default. However, we only need one private route table (for the Application Tier subnets). This is where clear naming conventions can immensely help guard against confusion, so let’s navigate to our route tables and clean things up a bit.
Select any one of the private route tables and adjust the name to something like ‘project1vpc-rtb-private1.’ This will be our private route table. Now we can associate this table with all four private subnets (-subnet-private1, -subnet-private2, -subnet-private-3, -subnet-private4)


 

Tier 1: Web tier (Frontend)
The Web Tier, also known as the ‘Presentation’ tier, is the environment where our application will be delivered for users to interact with. From this is where we will launch our web servers that will host the frontend of our application.
What we’ll build:
A web server launch template to define what kind of EC2 instances will be provisioned for the application.
An Auto Scaling Group (ASG) that will dynamically provision EC2 instances.
An Application Load Balancer (ALB) to help route incoming traffic to the proper targets.
1. Create a web server launch template
It’s time to create a template that will be used by our ASG to dynamically launch EC2 instances in our public subnets.
In the EC2 console, navigate to ‘Launch templates’ under the ‘Instances’ sidebar menu. We’re going to create a new template called ‘project1-webServer’ with the following provisions:
AMI: Amazon 2 Linux
Instance type: t2.micro (1GB – Free Tier)
A new or existing key pair
We’re not going to specify subnets, but we will create a new security group with inbound SSH, HTTP, and HTTPS rules. Make sure the proper VPC is selected.

 
Under ‘Advanced details > User data,’ we need to paste in our script that installs an Apache web server and a basic HTML web page.
 
2. Create an Auto scaling group (ASG)
To ensure high availability for the web app and limit single points of failure, we will create an ASG that will dynamically provision EC2 instances, as needed, across multiple AZs in our public subnets.
Navigate to the ASG console from the sidebar menu and create a new group. The ASG will use the project1-webServer-template launch template we set up in the previous step.
Select the vpc along with the two public subnets.
3. Application load balancer (ALB)
We’ll need an ALB to distribute incoming HTTP traffic to the proper targets (our EC2s). The ALB will be named, ‘project1-webServer-alb.’ We want this ALB to be ‘Internet-facing,’ so it can listen for HTTP/S requests.
The ALB needs to ‘listen’ over HTTP on port 80 and a target group that routes to our EC2 instances.
We’ll also add a dynamic scaling policy that tells the ASG when to scale up or down EC2 instances. For this build, we’ll monitor the CPU usage and create more instances when the usage is above 50% (feel free to use whatever metric is appropriate for your application).
 
Group size
have to set a minimum and maximum number of instances the ASG can provision:
Desired capacity: 2
Minimum capacity: 2
Maximum capacity: 5
Review the ASG settings and create the group!
Once the ASG is fully initialized, we can go to our EC2 dashboard and see that two EC2 instances have been deployed.
To see if our ALB is properly routing traffic, let’s go to its public DNS. We should be able to access the website we implemented when creating our EC2 launch template.

Tier 2: Application tier (Backend)
The Application Tier is essentially where the heart of our app lives. This is where the source code and core operations send/retrieve data to/from the Web and Database tiers.
The structure is very similar to the Web Tier but with some minor additions and considerations.
What we will build:
A launch template to define the type of EC2 instances.
An Auto Scaling Group (ASG) to dynamically provision EC2 instances.
An Application Load Balancer (ALB) to route traffic from the Web tier.
1. Create an application server launch template
This template will define what kind of EC2 instances our backend services will use, so let’s create a new template called, ‘project1-appServer-template.’
We will use the same settings as the project1-webServer-template (Amazon 2 Linux, t2.micro-1GB, same key pair).
Our security group settings are where things will differ. Remember, this is a private subnet, where all of our application source code will live. We need to take precautions so it cannot be accessible from the outside.
We want to allow ICMP–IPv4 from the project1-webServer-sg, which allows us to ping the application server from our web server.

2. Create an Auto Scaling Group (ASG)
Similar to the Web Tier, we’ll create an ASG from the project1-appServer-template called, ‘project1-appServer-asg.’
Make sure to select the vpc and the 2 private subnets (subnet-private1 and subnet-private2).
3. Application Load Balancer (ALB)
Now we’ll create another ALB that routes traffic from the Web Tier to the Application Tier. We’ll name it ‘project1-appServer-alb.’
This time, we want the ALB to be ‘Internal,’ since we’re routing traffic from our Web Tier, not the Internet.

The application servers will eventually need to access the database, so we need to make sure the mySQL package is installed on each instance.
In the ‘User data’ field under ‘Advanced details,’ paste in this script:
#!/bin/bash
sudo yum install mysql -y

Once the key pair is added to the Agent, SSH into the appserver.
And then SSH into our app server (remember, we need the private IPv4 address).
Success!
We’ve successfully built the Application Tier architecture for our application! Remember, this is the ‘Backend’ layer, where our source code lives and backend operations send/retrieve data to/from the Web Tier and Database Tier.

Tier 3: Database tier (Data storage & retrieval)
Almost there! Now it's time to build the last tier of our application architecture: the Database. Every application needs a way to store important data, such as user login info, session data, transactions, application content, etc… Our application servers need to be able to read and write to databases to perform necessary tasks and deliver proper content/services to the Web Tier and users.
We are going to use a Relational Database Service (RDS) that uses MySQL.
What we’ll build:
A database security group that allows outbound and inbound mySQL requests to and from our app servers.
A DB subnet group to ensure the database is created in the proper subnets.
An RDS database with MySql.
1. Create a database security group
Our application servers need a way to access the database, so let’s first create a security group that allows inbound traffic from the application servers.
Create a new security group called, ‘poj1-database-sg.’ Make sure the appropriate vpc is selected.

2. Create a DB subnet group
In the RDS console, under the ‘Subnet groups’ sidebar menu, create a new subnet group called, ‘poj1-database-sg.’ Make sure the appropriate vpc is selected.
 
Remember from our diagram, we want the database located in -subnet-private3 and span multiple AZs and subnets, if necessary.
Select our two AZs (ap-southeast-2a and ap-southeast-2b) and our private subnets (-subnet-private3 and -subnet-private4). Unfortunately, the selection dropdown, doesn't provide the subnet names, so we might have to navigate back to our main Subnets dashboard to get the right ids.
 
We should already have mySQL installed on the server, so we can run this command:
mysql -h YOUR_DB_ENDPOINT -P 3306 -u YOUR_DB_USERNAME -p
When prompted, enter the password you chose when creating the DB.

Success!
It was quite the journey! I know it wasn’t easy, but we took it step-by-step and pulled through. We’ve successfully created a highly available, 3-tier application architecture.
                                                                       



