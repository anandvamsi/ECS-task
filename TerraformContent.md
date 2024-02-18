
## Create a terraform version file
```bash
#main.tf

terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "4.45.0"
    }
  }
}
```

## Create a provider.
```
#provider.tf

provider "aws" {
  region  = "us-west-2" #The region where the environment
  #is going to be deployed # Use your own region here
}
```
## Create a ecr cluster
```bash
#ecs-cluster.tf

resource "aws_ecr_repository" "app_ecr_repo" {
  name = "app-repo"
}
```

Note:: Create a IAM role "ecsTaskExecutionRole" with below policy
```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```
Trust relationship
```bash

{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": "ecs-tasks.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

## Create a task definition for ecs cluster
```bash
#ecs-taskdf.tf

resource "aws_ecs_task_definition" "app_task" {
  family                   = "app-first-task" # Name your task
  container_definitions    = <<DEFINITION
  [
    {
      "name": "app-first-task",
      "image": "2100XXXXX852.dkr.ecr.us-west-2.amazonaws.com/app-repo:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080
        }
      ],
      "memory": 512,
      "cpu": 256
    }
  ]
  DEFINITION
  requires_compatibilities = ["FARGATE"] # use Fargate as the launch type
  network_mode             = "awsvpc"    # add the AWS VPN network mode as this is required for Fargate
  memory                   = 512         # Specify the memory the container requires
  cpu                      = 256         # Specify the CPU the container requires
  execution_role_arn       = "arn:aws:iam::21XXXXXX2:role/ecsTaskExecutionRole"
}
```




## Create a SG

```bash
#sg.tf
locals {
vpc = "vpc-0956XXXXXX9c10b8e"

}

resource "aws_security_group" "security_group" {
 name   = "ecs-security-group"
 vpc_id = local.vpc

 ingress {
   from_port   = 8080
   to_port     = 8080
   protocol    = "tcp"
   self        = "false"
   cidr_blocks = ["10.0.0.0/8"]
   description = "any"
 }

 egress {
   from_port   = 0
   to_port     = 0
   protocol    = "-1"
   cidr_blocks = ["0.0.0.0/0"]
 }


}
```

## Create a ecs-ALB
```bash
#ecs-alb.tf
resource "aws_alb" "application_load_balancer" {
  name               = "load-balancer-dev" #load balancer name
  internal           = true
  load_balancer_type = "application"
  subnets = [ # Referencing the default subnets
    "subnet-XXXXXXX5023",
    "subnet-047c8caXXXXd26"
  ]
  # security group
  security_groups = ["${aws_security_group.security_group.id}"]
}
```
## Create a target group

```bash
#target.tf
resource "aws_lb_target_group" "target_group" {
  name        = "target-group"
  port        = 8080
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = "vpc-095f761a169c10b8e" # default VPC
}

resource "aws_lb_listener" "listener" {
  load_balancer_arn = "${aws_alb.application_load_balancer.arn}" #  load balancer
  port              = "8080"
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = "${aws_lb_target_group.target_group.arn}" # target group
  }
}
```

## Create a service
```bash
#ecs-services.tf
resource "aws_ecs_service" "app_service" {
  name            = "app-first-service"     # Name the service
  cluster         = "${aws_ecs_cluster.my_cluster.id}"   # Reference the created Cluster
  task_definition = "${aws_ecs_task_definition.app_task.arn}" # Reference the task that the service will spin up
  launch_type     = "FARGATE"
  desired_count   = 3 # Set up the number of containers to 3

  load_balancer {
    target_group_arn = "${aws_lb_target_group.target_group.arn}" # Reference the target group
    container_name   = "${aws_ecs_task_definition.app_task.family}"
    container_port   = 8080 # Specify the container port
  }

  network_configuration {
    subnets          = ["subnet-047c8caae88d24be6","subnet-0bc434d8c9f250263"]
    assign_public_ip = false     # Provide the containers with public IPs
    security_groups  = ["${aws_security_group.security_group.id}"] # Set up the security group
  }
}
```

