## Create a Security group
## Create a load balancer
## ASG and Launch template
## ECS- Task def, and link to ASG

#create a SG 
-----------
```bash
resource "aws_security_group" "security_group" {
 name   = "ecs-security-group"
 vpc_id = aws_vpc.main.id

 ingress {
   from_port   = 8080
   to_port     = 8080
   protocol    = -1
   self        = "false"
   cidr_blocks = ["0.0.0.0/0"]
   description = "any"
 }

 egress {
   from_port   = 0
   to_port     = 0
   protocol    = "-1"
   cidr_blocks = ["0.0.0.0/0"]
 }
````

## Create a launch template
```bash
TERRAFORM
How to Deploy an AWS ECS Cluster with Terraform [Tutorial]

How to Deploy an AWS ECS Cluster with Terraform [Tutorial]
Sumeet Ninawe
29 Aug 2023
·
19 min read
Deploying an AWS ECS Cluster of EC2 Instances with Terraform
The Elastic Container Service (ECS) from Amazon Web Services (AWS) is a fully-managed cloud Container Orchestration Service. It runs multiple Docker containers on the cluster using AWS EC2 instances. Optionally it is also possible to run these containers on Fargate.

The more AWS ECS Clusters we deploy, the more complex the infrastructure management becomes. Especially when deploying and scaling a large cluster, the process becomes time-consuming as it involves a lot of repetitive tasks that lead to human errors, creating configuration drifts and increasing the risk of security breaches.

Using Terraform is the perfect solution to simplify the deployment of the AWS ECS Cluster. Terraform offers an automated way to manage AWS ECS Clusters, making the deployment process consistent and repeatable.

In this post, we will focus on how to set up an ECS cluster of EC2 instances using Terraform.

We will cover

How to deploy a service on ECS clusters
ECS deployment with Terraform – Overview
How to set up ECS with Terraform – Example
1. Setting up the VPC
2. Configuring the EC2 instances
3. Configuring the ECS cluster
4. Testing the ECS deployment
How to deploy a service on ECS clusters
There are mainly two ways of deploying a service on ECS clusters – Fargate and EC2 instances. This depends on the underlying infrastructure used to run the container workloads of any ECS service.

AWS Fargate is a more cloud-native approach where the compute instances are automatically managed by AWS.

Running the ECS service on EC2 instances provides more control over the infrastructure. This also requires taking additional steps to set up the EC2 instances and auto-scaling group, networking, etc.

An ECS service generally consists of the components shown in the diagram below.

ecs task definition terraform
When an ECS Cluster is created, before any service is deployed, we have to provide the cluster with capacity providers.

In this blog post, the capacity providers are the EC2 instances where the scaling is managed by ASG.  The service can then be deployed once the capacity providers or the capacity providing EC2 instances are registered with the ECS cluster.

An ECS service consists of multiple tasks. Each task is created based on the task definition provided by the service.

A task definition is a template that describes the source of the application image, resources required in terms of CPU and memory units, container and host port mapping, and other critical information.

Apart from the task definition, ECS service also provides the number of container instances to be created. This automatically creates the tasks, which are then assigned a target EC2 instance infrastructure where the containers are run. The running containers serve the incoming requests from the application load balancer (ALB) which is deployed in front of the EC2 instances in a VPC.

Why use ECS over EC2?
ECS and EC2 are services provided by AWS for running applications in the cloud. Although at a high level, they seem to host workloads, they serve different purposes and are suited for different types of workloads.

Some of the reasons for choosing ECS over EC2 are described below:

Containerization: ECS is designed for running containers either on Fargate or EC2 instances, which provide a scalable way to package and deploy applications. Containers offer better resource utilization, faster startup times, and improved isolation compared to traditional virtual machines used in EC2. If you’re using containerized applications, ECS provides a managed environment optimized for running and orchestrating containers.
Scalability and Elasticity: ECS allows you to easily scale your applications based on demand. It integrates with AWS Auto Scaling, which can automatically adjust the number of containers based on predefined scaling policies and placement constraints.
Orchestration: ECS provides built-in orchestration capabilities through integration with AWS Fargate or EC2 launch types. With Fargate, we don’t have to provision or manage any EC2 instances, as AWS takes care of the infrastructure for you. This is also a great option and allows us to focus solely on our applications. EC2 launch type offers more control and flexibility if we prefer managing the underlying EC2 instances ourself.
Monitoring: ECS provides a centralized management console and CLI tools for deploying and managing containers. It also integrates with AWS CloudWatch, allowing us to collect and analyze metrics, monitor logs, and set up alarms for our containerized applications. EC2 instances require separate management and monitoring configurations.
Cost Optimization: ECS can help optimize costs by allowing us to run containers on-demand without the need for permanent EC2 instances. With AWS Fargate, we pay only for the resources allocated to our containers while they are running, which is more cost-effective compared to maintaining a fleet of EC2 instances. Read more about AWS cost optimization.
It’s important to note that the choice between ECS and EC2 depends on our specific requirements, workload characteristics, and familiarity with containerization. While ECS offers benefits for container-based applications, EC2 still provides more flexibility for running traditional workloads that don’t require containerization.

ECS deployment with Terraform - Overview
The diagram below shows the outcome of ECS deployment using Terraform.

It shows how an ECS cluster is set up on EC2 instances spread across multiple availability zones within a VPC. It also includes several details like Application Load Balancers (ALB), auto-scaling group (ASG), ECS Capacity provider, ECS service, etc.

Note that there are other aspects not represented in the diagram, like ECR, Route tables, task definition, etc., which we will also cover.

terraform aws ecs
We will break this infrastructure down into three parts. The list below provides an overview at a high level of all the steps we will be taking to achieve our goal of hosting ECS clusters on EC2 instances. As we proceed, we will also write the corresponding Terraform configuration.

VPC setup – In this part, we will explain and implement the basic VPC and networking setup required. We will implement subnets, security groups, and route tables, to access the hosted service from the internet as well as to SSH into our EC2 instances. 
EC2 setup – In this part, we will explain and implement auto-scaling groups, application load balancer, and EC2 instances, across the two availability zones. We will also cover the setup in details required to host ECS container instances on our EC2 machines.
ECS setup – Finally, we do a step-by-step creation of the ECS cluster by creating ECS clusters, services, tasks, and capacity providers. We will also take a look at the application image which will be used to run on this ECS cluster. Additionally, we will also show how to create task definitions along with various Terraform resource attributes which enable the end-to-end deployment of the service.
Note: The final Terraform configuration is saved in this Github repository.

Before moving ahead, it is highly recommended to follow these steps hands-on.

How to setup ECS with Terraform - Example
Before we start, you should have Terraform installed locally. We will not go through the details of setting up the providers, AWS credentials, and AWS CLI.

Let’s go through the steps to deploy ECS using Terraform.

1. Setting up the VPC
To begin with, let us first establish our isolated network by defining the VPC and related components. This is important as we need to create multiple container instances of our application so that the load balancer is able to distribute the requests evenly.

The diagram below displays the target VPC design we would achieve using Terraform.

aws ecs service terraform
Create a file named vpc.tf in the project repository and add the code below.

You can optionally include all the Terraform code in the same file (e.g., in main.tf). However, I prefer to keep separate files to manage the IaC easily. 

Step 1: Define our VPC
The code below creates a VPC with a given CIDR range. You are free to choose a CIDR range.

As an example, we use 10.0.0.0/16 in this case and name our VPC as “main”.

resource "aws_vpc" "main" {
 cidr_block           = var.vpc_cidr
 enable_dns_hostnames = true
 tags = {
   name = "main"
 }
}
Step 2: Add 2 subnets
Create two subnets in different availability zones to place our EC2 instances. We have used “cidrsubnet” Terraform function to dynamically calculate the CIDR range based on the VPC’s CIDR.

Note that we are using different availability zones in the Frankfurt region for both subnets.

resource "aws_subnet" "subnet" {
 vpc_id                  = aws_vpc.main.id
 cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, 1)
 map_public_ip_on_launch = true
 availability_zone       = "eu-central-1a"
}

resource "aws_subnet" "subnet2" {
 vpc_id                  = aws_vpc.main.id
 cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, 2)
 map_public_ip_on_launch = true
 availability_zone       = "eu-central-1b"
}
Step 3: Create internet gateway (IGW)
resource "aws_internet_gateway" "internet_gateway" {
 vpc_id = aws_vpc.main.id
 tags = {
   Name = "internet_gateway"
 }
}
Step 4: Create a Route table and associate the same with subnets
This route table enables internet communication of both the subnets, making them public.

resource "aws_route_table" "route_table" {
 vpc_id = aws_vpc.main.id
 route {
   cidr_block = "0.0.0.0/0"
   gateway_id = aws_internet_gateway.internet_gateway.id
 }
}

resource "aws_route_table_association" "subnet_route" {
 subnet_id      = aws_subnet.subnet.id
 route_table_id = aws_route_table.route_table.id
}

resource "aws_route_table_association" "subnet2_route" {
 subnet_id      = aws_subnet.subnet2.id
 route_table_id = aws_route_table.route_table.id
}
Step 5: Create a security group along with ingress and egress rules
Both ingress and egress rules of the security group allow inbound and outbound access for any protocol, via any port. This is not the best practice and should only be done for working through this example. Tighter rules should be implemented when in production.

resource "aws_security_group" "security_group" {
 name   = "ecs-security-group"
 vpc_id = aws_vpc.main.id

 ingress {
   from_port   = 0
   to_port     = 0
   protocol    = -1
   self        = "false"
   cidr_blocks = ["0.0.0.0/0"]
   description = "any"
 }

 egress {
   from_port   = 0
   to_port     = 0
   protocol    = "-1"
   cidr_blocks = ["0.0.0.0/0"]
 }
}
At this point, this is all we need as far as the VPC and networking is concerned. You can go ahead and provision these resources now or proceed with the next section.

The VPC setup is quite straightforward. We will not go deep into exploring the VPC creation using Terraform, as that is not the topic of this blog post. For more information on that, check out How to Build AWS VPC using Terraform.

This gives us the networking basics to proceed to the next step.

2. Configuring the EC2 instances
In this section, we are focusing on provisioning the auto-scaling group, defining the EC2 instance template used to host the ECS containers, and provisioning the application load balancer in the VPC created in the previous section.

Below you can see the updated diagram.

 

terraform ecs cluster example
Create another file named ec2.tf, and follow the steps below. 

Step 1: Create an EC2 launch template
A launch template, as the name suggests, defines the template used by the auto-scaling group to provision and maintain a desired/required number of EC2 instances in a cluster.

Launch templates define various characteristics of the EC2 instance:

Image: We use Amazon Linux image with CPU architecture as AMD.
Type: Size of the instance. We use “t3.micro”.
The size of the instance is decided by the system resources consumed by the container. In our case, we are hosting a “Docker Getting Started” image which does not consume a lot of resources. More about the image will be covered later in the post.
Key name: Specify the name of the key to be able to SSH into these instances from our local machines. You can either create a key using a separate Terraform resource block. In this case, I am just using a key that I have already generated.
Security group: The security group to be associated with the EC2 instance. Associate the same security group we created in the previous section.
IAM instance profile: This is very important. Without this, the EC2 instances will not be able to access the ECS service in AWS. “ecsInstanceRole” is a predefined role available in all the AWS accounts. However, if you want to use a custom role, make sure it can access the ECS service.
User data: This is also very important. The “ecs.sh” file contains a command to create an environment variable in “/etc/ecs/ecs.config” file on each EC2 instance that will be created. Without setting this, the ECS service will not be able to deploy and run containers on our EC2 instance.
resource "aws_launch_template" "ecs_lt" {
 name_prefix   = "ecs-template"
 image_id      = "ami-062c116e449466e7f"
 instance_type = "t3.micro"

 key_name               = "<keyname>"
 vpc_security_group_ids = [aws_security_group.security_group.id]
 iam_instance_profile {
   name = "ecsInstanceRole"
 }

 block_device_mappings {
   device_name = "/dev/xvda"
   ebs {
     volume_size = 30
     volume_type = "gp2"
   }
 }

 tag_specifications {
   resource_type = "instance"
   tags = {
     Name = "ecs-instance"
   }
 }

 user_data = filebase64("${path.module}/ecs.sh")
}
```
## contents of ecs.sh
```bash
#!/bin/bash
echo ECS_CLUSTER=<name of cluster> >> /etc/ecs/ecs.config
```

## ASG
```bash
resource "aws_autoscaling_group" "ecs_asg" {
 vpc_zone_identifier = [aws_subnet.subnet.id, aws_subnet.subnet2.id]
 desired_capacity    = 2
 max_size            = 3
 min_size            = 1

 launch_template {
   id      = aws_launch_template.ecs_lt.id
   version = "$Latest"
 }
 }
```
## ASG-Health-check
```bash
resource "aws_lb" "ecs_alb" {
 name               = "ecs-alb"
 internal           = false
 load_balancer_type = "application"
 security_groups    = [aws_security_group.security_group.id]
 subnets            = [aws_subnet.subnet.id, aws_subnet.subnet2.id]

 tags = {
   Name = "ecs-alb"
 }
}

resource "aws_lb_listener" "ecs_alb_listener" {
 load_balancer_arn = aws_lb.ecs_alb.arn
 port              = 8080
 protocol          = "HTTP"

 default_action {
   type             = "forward"
   target_group_arn = aws_lb_target_group.ecs_tg.arn
 }
}

resource "aws_lb_target_group" "ecs_tg" {
 name        = "ecs-target-group"
 port        = 8080
 protocol    = "HTTP"
 target_type = "ip"
 vpc_id      = aws_vpc.main.id

 health_check {
   path = "/"
 }
}
```

## Create ECS cluster
```bash
resource "aws_ecs_cluster" "ecs_cluster" {
 name = "ecs-cluster"
}
```

## Create a capacity providers
```bash
resource "aws_ecs_capacity_provider" "ecs_capacity_provider" {
 name = "test1"

 auto_scaling_group_provider {
   auto_scaling_group_arn = aws_autoscaling_group.ecs_asg.arn

   managed_scaling {
     maximum_scaling_step_size = 1000
     minimum_scaling_step_size = 1
     status                    = "ENABLED"
     target_capacity           = 3
   }
 }
}

resource "aws_ecs_cluster_capacity_providers" "example" {
 cluster_name = aws_ecs_cluster.ecs_cluster.name

 capacity_providers = [aws_ecs_capacity_provider.ecs_capacity_provider.name]

 default_capacity_provider_strategy {
   base              = 1
   weight            = 100
   capacity_provider = aws_ecs_capacity_provider.ecs_capacity_provider.name
 }
}
```
