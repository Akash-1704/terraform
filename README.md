**Terraform AWS Auto Scaling Group Setup**

This repository contains a Terraform configuration file (main.tf) that sets up an Auto Scaling Group (ASG) on AWS. This setup ensures high availability and scalability of your application by automatically adjusting the number of instances in response to traffic patterns.
Prerequisites

    Terraform installed on your local machine.
    AWS credentials configured on your machine. You can set these up using the AWS CLI.
    Access to an AWS account with permissions to create resources such as VPCs, subnets, EC2 instances, and load balancers.

Configuration Overview
Providers

The configuration uses the AWS provider to interact with AWS resources:

hcl

provider "aws" {
  region = "ap-south-1"
}

Variables

A variable server_port is defined to set the port for the HTTP server:

hcl

variable "server_port" {
  description = "The port the server will use for HTTP requests"
  type        = number
  default     = 8080
}

Data Sources

Data sources are used to fetch information about existing AWS resources:

hcl

data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

Resources
Launch Configuration

Defines a launch configuration for the Auto Scaling Group:

hcl

resource "aws_launch_configuration" "example" {
  image_id        = "ami-0f5ee92e2d63afc18"
  instance_type   = "t2.micro"
  security_groups = [aws_security_group.instance.id]
  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p ${var.server_port} &
              EOF
  lifecycle {
    create_before_destroy = true
  }
}

Auto Scaling Group

Creates an Auto Scaling Group with the specified configurations:

hcl

resource "aws_autoscaling_group" "example" {
  launch_configuration = aws_launch_configuration.example.name
  vpc_zone_identifier  = data.aws_subnets.default.ids
  target_group_arns = [aws_lb_target_group.asg.arn]
  health_check_type = "ELB"
  min_size = 2
  max_size = 10
  tag {
    key                 = "Name"
    value               = "terraform-asg-example"
    propagate_at_launch = true
  }
}

Load Balancer

Configures an Application Load Balancer (ALB):

hcl

resource "aws_lb" "example" {
  name               = "terraform-asg-example"
  load_balancer_type = "application"
  subnets            = data.aws_subnets.default.ids
  security_groups    = [aws_security_group.alb.id]
}

Security Groups

Defines security groups for the load balancer and instances:

hcl

resource "aws_security_group" "alb" {
  name = "terraform-example-alb"
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "instance" {
  name = "terraform-example-instance"
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

Load Balancer Listeners and Target Groups

Configures listeners and target groups for the load balancer:

hcl

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.example.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "404: page not found"
      status_code  = 404
    }
  }
}

resource "aws_lb_listener_rule" "asg" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 100
  condition {
    path_pattern {
      values = ["*"]
    }
  }
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.asg.arn
  }
}

resource "aws_lb_target_group" "asg" {
  name     = "terraform-asg-example"
  port     = var.server_port
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id
  health_check {
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 15
    timeout             = 3
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

Bastion Host

Creates a bastion host instance:

hcl

resource "aws_instance" "bastion" {
  ami                 = "ami-0f5ee92e2d63afc18"
  instance_type       = "t2.micro"
  vpc_security_group_ids = [aws_security_group.instance.id]
  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF
  user_data_replace_on_change = true
  tags = {
    Name = "terraform-example"
  }
}

Outputs

Defines outputs for the Terraform configuration:

hcl

output "alb_dns_name" {
  value       = aws_lb.example.dns_name
  description = "The domain name of the load balancer"
}

Usage

    Clone the repository:

    sh

git clone https://github.com/your-repo.git
cd your-repo

Initialize Terraform:

sh

terraform init

Review the execution plan:

sh

terraform plan

Apply the configuration:

sh

terraform apply

Retrieve the Load Balancer DNS:
After the configuration is applied, you can get the DNS name of the load balancer using:

sh

    terraform output alb_dns_name

Notes

    Ensure your AWS credentials are correctly configured to avoid authentication issues.
    Modify the main.tf file as needed to customize the setup to your requirements.
    Monitor the Auto Scaling Group to ensure it is scaling as expected based on your application's traffic.

By following these instructions, you will set up a scalable, highly available infrastructure for your application on AWS using Terraform.
