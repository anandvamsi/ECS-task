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
