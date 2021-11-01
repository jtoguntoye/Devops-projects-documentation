# Automate Infrastructure with IAC using Terraform - Part2

Infrastructure:

![project-15-infrastructure IMG](https://user-images.githubusercontent.com/23315232/135461812-e94e31b1-526b-4950-b82a-e910ee53773c.png)
Credits: Darey.io

Note:
This documentation is a continuation of project 16 using Terraform to provision the AWS architecture shown above. I this project we will be creating the 
nginx server, bastion webservers, launch templates autoscaling groups, RDS and EFS instance for the company's architecture.

## Create private subnets
- Dynamically create 4 private subnets. Two of these will hold the web servers while the other two will hold the datalayer(EFS and RDS instances)
  - Make sure you use variables or length() function to determine the number of AZs
  - Use variables and cidrsubnet() function to allocate vpc_cidr for subnets
  - Keep variables and resources in separate files for better code structure and readability
  
  ```
  # create private subnets for web servers
  resource "aws_subnet" "private-A" {
      count = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names): var.preferred_number_of_private_subnets
      vpc_id = aws_vpc.main.id
      cidr_block = cidrsubnet(var.vpc_cidr, 8, count.index + 3)
      availability_zone = data.aws_availability_zones.available.names[count.index]    
      tags = merge(
            var.tags,
            {
             Name = format("privateSubnet-%s", count.index + 1)
            }
        )
  
  }    

  # create private subnets for data storage layer
  resource "aws_subnet" "private-B" {
      count = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names): var.preferred_number_of_private_subnets
      vpc_id = aws_vpc.main.id
      cidr_block = cidrsubnet(var.vpc_cidr, 8, count.index + 5)
      availability_zone = data.aws_availability_zones.available.names[count.index]    
      tags = merge(
            var.tags,
            {
             Name = format("PrivateSubnet-%s", count.index + 3)
            }
        )
  
  }  
  
  ```
  
  ### Tagging
  - Tagging helps manage resources more efficiently. With tags resources are more organized in virtual groups. 
  - Tagging allows you to easily search for resources.
  - Billing teams can easily generate reports and determine what each part of the infrastructure costs.
  
  - Add  multiple tags as a default tag for resources.
  - Declare a `tag` variable in the `variable.tf` file and define its value in the `terraform.tfvars` file respectively as shown below:
  
  ```
  variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
  }
 
   tags = {
  Environment = "Dev"
  Owner-Email = "joeloguntoye@gmail.com"
  Managed-By  = "Terraform"
  Billing-Account = "384543527682"    
  }
  


## Create Internet Gateway
- Create a new file `internet_gateway.tf` and define an internet gateway resource:
```
#create internet gateway 
resource "aws_internet_gateway" "ig" {
    vpc_id = aws_vpc.main.id

    tags = merge(
        var.tags,
        {
            Name = format("%s-%s", aws_vpc.main.id, "IG")
        }
    )
  
}
```
## Create a NAT gateway
- Create 1 NAT Gateways and 1 Elastic IP (EIP) addresses
- The elastic IP address will be attached to the NAT gateway. Both the EIP address and the NAT gateway depend on the internet gateway 
This is reflected in the creation of the NAT gateway. A `depends-on` atttribute is defined for the EIP and  the NAT gateway resource to indicate to Terraform that the internet gateway must be created first

```
#create eip for nat gateway
resource "aws_eip" "nat_eip" {
    vpc = true
    depends_on = [aws_internet_gateway.ig]
    tags = merge (
        var.tags,
        {
            Name = format("%s-EIP", var.eip_name)
        }
    )
}

#create nat gateway
resource "aws_nat_gateway" "nat" {
    allocation_id = aws_eip.nat_eip.id
    subnet_id = element(aws_subnet.public[*].id, 0)
    depends_on = [aws_internet_gateway.ig]
    
    tags = merge (
        var.tags,
        {
            Name = format("%s", var.nat_gw)
        }
    )
  
}
```
## Create AWS Routes
- Create a new file named `route-tables.tf`
- Create public and private route tables and associate the appropriate public and private subnets respectively. Also, the internet gateway created earlier is 
specified for the public route to allow access from the internet for subnets in the public route table
- The NAT gateway defined earlier is attached to the private route table to allow the webservers forward traffic to the internet.

```
#create private routes table 
resource "aws_route_table" "private-rtb" {
    vpc_id = aws_vpc.main.id

    tags = merge(
        var.tags,
        {
            Name = format("%s-private-route-table", var.private-rtb-name)
        }
    )
}

# create a route in the private route table
resource "aws_route" "private-rtb-route" {
    route_table_id = aws_route_table.private-rtb.id
    destination_cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id

}

#associate all private-A subnets to the private route table
resource "aws_route_table_association" "private-subnets-assoc-A" {
  count = length(aws_subnet.private-A[*].id)
  subnet_id = element(aws_subnet.private-A[*].id, count.index)
  route_table_id = aws_route_table.private-rtb.id
}

#associate all private-B subnets to the private route table
resource "aws_route_table_association" "private-subnets-assoc-B" {
  count = length(aws_subnet.private-B[*].id)
  subnet_id = element(aws_subnet.private-B[*].id, count.index)
  route_table_id = aws_route_table.private-rtb.id
}

#create route table for the public subnets
resource "aws_route_table" "public-rtb" {
  vpc_id = aws_vpc.main.id

  tags = merge (
      var.tags,
      {
          Name = format("%s-public-route-table", var.public-rtb-name)
      }
  )
}

#create route in the public route table
resource "aws_route" "public-rtb-route" {
  route_table_id = aws_route_table.public-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.ig.id
}

# associate all the public subnets with the public route table
resource "aws_route_table_association" "public-subnets-assoc" {
    count = length(aws_subnet.public[*].id)
    subnet_id = element(aws_subnet.public[*].id, count.index)
    route_table_id = aws_route_table.public-rtb.id
}

```

## AWS Identity and Access Management
- We want to pass IAM Role to EC2 instances to give them access to some specific resources.

- First create `AssumeRole`. AssumeRole in AWS using Security Token Service (STS) API returns a set of temporary security credentials that you can use to 
access AWS resources. These temporary credentials consist of an access key ID, a secret access key, and a security token. Typically, you use AssumeRole within your account or for cross-account access
- Create a new file named `roles.tf` to define the IAM role for EC2  instances to assume.
```
# create Iam role to be used by EC2-instance
resource "aws_iam_role" "ec2_instance_role" {
  name = "ec2_instance_role"
  assume_role_policy = jsonencode({
      Version = "2012-10-17"
      Statement = [
        {
              Sid = ""
              Action = "sts:AssumeRole"
              Effect = "Allow"
              Principal = {
                  Service = "ec2.amazonaws.com"
              }
        },
      ]
  })

  tags = merge (
      var.tags,
      {
          Name = "aws assume role"
      }
  )
}

```
- The `aws_iam_role` created above includes an `assume_role_policy`. This policy specifies which entity can assume the role we defined.

- Next create IAM policy for this role. The Iam Policy specifies what an entity is allowed to do when its assumes this role.

```
resource "aws_iam_policy" "policy" {
  name        = "ec2_instance_policy"
  description = "A test policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "ec2:Describe*",
        ]
        Effect   = "Allow"
        Resource = "*"
      },
    ]

  })

  tags = merge(
    var.tags,
    {
      Name =  "aws assume policy"
    },
  )

}
```
- Next attach the Iam policy to the IAM role
```
# attach the iam_policy to the iam role 
resource "aws_iam_role_policy_attachment" "test_attach" {
    role = aws_iam_role.ec2_instance_role.name
    policy_arn = aws_iam_policy.policy.arn
}
```
- Next, create an instance profile.  We Use an instance profile to pass an IAM role to an EC2 instance
```
#create an instance profile to pass an iam role to an ec2 instance
resource "aws_iam_instance_profile" "instProfile" {
    name = "aws_instance_profile_test"
    role = aws_iam_role.ec2_instance_role.name
}
```

## Create Security Groups
- We'll define all the security groups and security group rules in one file, then reference each resource within each resource that needs it.
- Create a file named `security.tf` and define the security groups

```
# security group for alb, to allow access from anywhere for  http and https traffic
resource "aws_security_group" "ext-alb-sg" {
  name = var.ext-alb-sg-name
  vpc_id = aws_vpc.main.id
  description = "Allow TLS inbound traffic"

  ingress =  [
      {
    description = "http"
    from_port = 80
    protocol = "tcp"
    to_port = 80
    cidr_blocks = [ "0.0.0.0/0"]
    ipv6_cidr_blocks = []
    prefix_list_ids = []
    security_groups = []
    self = false
    },

    {
    description = "https"
    from_port = 443
    protocol = "tcp"
    to_port = 443
    cidr_blocks = [ "0.0.0.0/0"]
    ipv6_cidr_blocks = []
    prefix_list_ids = []
    security_groups = []
    self = false
    } 
  ]

   
  
  egress = [
      {
      description = "outgoing"
      from_port = 0
      to_port = 0
      protocol = "-1"
      cidr_blocks = ["0.0.0.0/0"]
      ipv6_cidr_blocks = []
      prefix_list_ids = []
      security_groups = []
      self = false
  }
  ]

  tags = merge (
      var.tags,
      {
        Name = "ext-alb-sg"  
      },
  )
}

# security group for Bastion, to allow access from your IP to the bastion host
resource "aws_security_group" "bastion-sg" {
    name = "bastion-sg"
    vpc_id = aws_vpc.main.id
    description = "Allow incoming ssh connection"

    ingress = [
        {
        description = "SSH"
        from_port = 22
        to_port = 22
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        cidr_blocks = [ "0.0.0.0/0"]
        ipv6_cidr_blocks = []
        prefix_list_ids = []
        security_groups = []
        self = false
        }
    ]    
    
    egress = [
     {
        description = "outgoing"
        from_port = 22
        to_port = 22
        protocol = "-1"
        cidr_blocks = [ "0.0.0.0/0"]
        ipv6_cidr_blocks = []
        prefix_list_ids = []
        security_groups = []
        self = false
     }
   ]

    tags = merge (
        var.tags,
        {
            Name = "Bastion-sg"
        }
    )
  
}

# create security group for nginx reverse proxy, to allow access only from the external load balancer
# and Bastion instance
resource "aws_security_group" "nginx-sg" {
    name = "nginx-sg"
    vpc_id = aws_vpc.main.id

    egress = [
        {
        description = "outgoing"    
        from_port = 0
        to_port = 0
        protocol = -1
        cidr_blocks = [ "0.0.0.0/0"]
        ipv6_cidr_blocks = []
        prefix_list_ids = []
        security_groups = []
        self = false
    } 
    ]

    tags = merge (
        var.tags,
        {
            Name = "nginx-SG"
        }
    )    
}

# ingress rule to attach to the nginx sg
resource "aws_security_group_rule" "inbound-nginx-https" {
    type = "ingress"
    from_port = 443
    to_port = 443
    protocol = "tcp"
    source_security_group_id = aws_security_group.ext-alb-sg.id
    security_group_id = aws_security_group.nginx-sg.id
}

#ingress rule to attach to the nginx sg to allow access from bastion
resource "aws_security_group_rule" "inbound-bastion-ssh" {
   type = "ingress"
   from_port = 22
   to_port = 22
   protocol = "tcp"
   source_security_group_id = aws_security_group.bastion-sg.id
   security_group_id = aws_security_group.nginx-sg.id
}

# security group for ialb, to have acces only from nginx reverser proxy server
resource "aws_security_group" "int-alb-sg" {
  name   = "my-alb-sg"
  vpc_id = aws_vpc.main.id

  egress =  [
    {
    description = "outgoing"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = []
    prefix_list_ids = []
    security_groups = []
    self = false

    }
  ]

  

  tags = merge(
    var.tags,
    {
      Name = "int-alb-sg"
    },
  )

}

resource "aws_security_group_rule" "inbound-ialb-https" {
     type = "ingress"
     from_port = 443
     to_port  = 443
     protocol = "tcp"
     source_security_group_id = aws_security_group.nginx-sg.id
     security_group_id = aws_security_group.int-alb-sg.id
}

# security group for webservers, to have access only from the internal load balancer and bastion instance
resource "aws_security_group" "webserver-sg" {
  name   = "my-asg-sg"
  vpc_id = aws_vpc.main.id

  egress = [
    {
    description = "outgoing"
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = []
    prefix_list_ids = []
    security_groups = []
    self = false
    }
  ]

  tags = merge(
    var.tags,
    {
      Name = "webserver-sg"
    },
  )

}

resource "aws_security_group_rule" "inbound-web-https" {
  type  = "ingress"
  from_port = 443
  to_port = 443
  protocol = "tcp"
  source_security_group_id = aws_security_group.int-alb-sg.id
  security_group_id = aws_security_group.webserver-sg.id
}

resource "aws_security_group_rule" "inbound-web-ssh" {
  type = "ingress"
  from_port = 22
  to_port = 22
  protocol = "tcp"
  source_security_group_id = aws_security_group.bastion-sg.id
  security_group_id = aws_security_group.webserver-sg.id
}

# security group for datalayer to allow traffic from webserver to nfs and mysql port 
#and bastion host to mysql port
resource "aws_security_group" "datalayer-sg" {
    name = "datalayer-sg"
    vpc_id = aws_vpc.main.id
    egress = [
        {
        description = "outgoing"
        from_port = 0
        to_port = 0
        protocol = "-1"
        cidr_blocks = ["0.0.0.0/0"]
        ipv6_cidr_blocks = []
        prefix_list_ids = []
        security_groups = []
        self = false
    }
    ]
    tags = merge (
        var.tags,
        {
            Name = "datalayer-sg"
        }
    )
}

resource "aws_security_group_rule" "inbound-nfs-webserver" {
  type = "ingress"
  from_port = 2049
  to_port = 2049
  protocol = "tcp"
  security_group_id = aws_security_group.datalayer-sg.id
  source_security_group_id = aws_security_group.webserver-sg.id
}

resource "aws_security_group_rule" "inbound-nfs-nginx" {
  type = "ingress"
  from_port = 2049
  to_port = 2049
  protocol = "tcp"
  security_group_id = aws_security_group.datalayer-sg.id
  source_security_group_id = aws_security_group.nginx-sg.id
}

resource "aws_security_group_rule" "inbound-mysql-webserver" {
  type = "ingress"
  from_port = 3306
  to_port = 3306
  protocol = "tcp"
  source_security_group_id = aws_security_group.webserver-sg.id
  security_group_id = aws_security_group.datalayer-sg.id
}

resource "aws_security_group_rule" "inbound-mysql-bastion" {
  type = "ingress"
  from_port = 3306
  to_port = 3306
  protocol = "tcp"
  source_security_group_id = aws_security_group.bastion-sg.id
  security_group_id = aws_security_group.datalayer-sg.id
}
```
## Create Amazon Certificate Manager
- Create a new file `cert.tf` to hold the Amazon-managed TLS/SSL certificate to be attached to the external load balancer. 
```
# create an Amazon TLS/SSL certificate , public zone and validate the certificate using DNS method

# create the certificate using a wildcard for all the domains created in kiff-web.space
resource "aws_acm_certificate" "kiff-web" {
    domain_name = "*.kiff-web.space"
    validation_method = "DNS"
}

# creating the hosted zone
data "aws_route53_zone" "kiff-web" {
    name = "kiff-web.space"
    private_zone = false
}

# selecting a validation method
resource "aws_route53_record" "kiff-web" {
    for_each = {
        for dvo in aws_acm_certificate.kiff-web.domain_validation_options : dvo.domain_name => {
            name = dvo.resource_record_name
            record = dvo.resource_record_value
            type = dvo.resource_record_type
        }
    }

    allow_overwrite = true
    name = each.value.name
    records = [each.value.record]
    ttl = 60
    type = each.value.type
    zone_id = data.aws_route53_zone.kiff-web.zone_id
}

# validate the certificate through DNS method
resource "aws_acm_certificate_validation" "kiff-web" {
    certificate_arn = aws_acm_certificate.kiff-web.arn
    validation_record_fqdns = [for record in aws_route53_record.kiff-web : record.fqdn]
}

# create record for tooling 
resource "aws_route53_record" "tooling" {
  zone_id = data.aws_route53_zone.kiff-web.id
  name = "tooling.kiff-web.space"
  type = "A"

  alias {
    name = aws_lb.ext-alb.dns_name
    zone_id = aws_lb.ext-alb.zone_id
    evaluate_target_health = true
  }
}

# create record for wordpress
resource "aws_route53_record" "wordpress" {
    name = "wordpress.kiff-web.space"
    zone_id = data.aws_route53_zone.kiff-web.zone_id
    type = "A"

    alias {
      name = aws_lb.ext-alb.dns_name
      zone_id = aws_lb.ext-alb.zone_id
      evaluate_target_health = true
    }
  
}
```
## Create External and Internal load balancers
- create a file named `alb.tf`. This file will hold the external and internal load balancers, target groups and listener rules for the load balancers
```
#create external load balancer
resource "aws_lb" "ext-alb" {
    name = "ext-alb"
    internal = false
    security_groups = [
        aws_security_group.ext-alb-sg.id,
    ]

    subnets = [
        aws_subnet.public[0].id,
        aws_subnet.public[1].id
    ]

    tags = merge (
        var.tags,
        {
            Name = "kiff-ext-alb"
        },
    )

    ip_address_type = "ipv4"
    load_balancer_type = "application"
}

# create target group for the external alb
resource "aws_lb_target_group" "nginx-tgt" {
  health_check {
    interval  = 10
    path = "/healthstatus"
    protocol = "HTTPS"
    timeout = 5
    healthy_threshold = 5
    unhealthy_threshold = 2
  }

  name = "nginx-tgt"
  port = 443
  protocol = "HTTPS"
 target_type = "instance"
 vpc_id = aws_vpc.main.id
}

# create a listener for the target group
resource "aws_lb_listener" "nginx-listener" {
  load_balancer_arn = aws_lb.ext-alb.arn
  port = 443
  protocol = "HTTPS"
  certificate_arn = aws_acm_certificate_validation.kiff-web.certificate_arn

  default_action {
    type = "forward"
    target_group_arn = aws_lb_target_group.nginx-tgt.arn
  }
}

# create an internal Application Load Balancer for the webservers

resource "aws_lb" "ialb" {
    name = "ialb"
    internal = true
    security_groups = [
        aws_security_group.int-alb-sg.id
    ]

    subnets = [
        aws_subnet.private-A[0].id,
        aws_subnet.private-A[1].id
    ]

    tags = merge (
        var.tags,
        {
            Name = "kiff-int-LB"
        },
    )
    
    ip_address_type = "ipv4"
    load_balancer_type = "application"
}

#--- create target group for internal LB ---

resource "aws_lb_target_group" "wordpress-tgt" {
    health_check {
      interval  = 10
      path = "/healthstatus"
      protocol = "HTTPS"
      timeout = 5
      healthy_threshold = 5
      unhealthy_threshold = 2
    }

    name = "wordpress-tgt"
    port = 443
    protocol = "HTTPS"
    target_type = "instance"
    vpc_id = aws_vpc.main.id
}

# target group for tooling
resource "aws_lb_target_group" "tooling-tgt" {
    health_check {
      interval = 10
      path = "/healthstatus"
      protocol = "HTTPS"
      timeout = 5
      healthy_threshold = 5
      unhealthy_threshold = 2
    }

    name = "tooling-tgt"
    port = 443
    protocol = "HTTPS"
    target_type = "instance"
    vpc_id = aws_vpc.main.id
}

# ---create a listener for the wordpress target group which will be the default --- 
resource "aws_lb_listener" "web_listener" {
 load_balancer_arn = aws_lb.ialb.arn
 port = 443
 protocol = "HTTPS"
 certificate_arn = aws_acm_certificate_validation.kiff-web.certificate_arn

 default_action {
   type = "forward"
   target_group_arn = aws_lb_target_group.wordpress-tgt.arn
 }  
}

#configure listener rule for tooling target

resource "aws_lb_listener_rule" "tooling-listener" {
    listener_arn = aws_lb_listener.web_listener.arn
    priority = 99

    action {
      type = "forward"
      target_group_arn = aws_lb_target_group.tooling-tgt.arn
    }

    condition {
      host_header {
          values = ["tooling.kiff-web.space"]
      }
    }
}

```

## Create Autoscaling groups
- Create a new file named `asg-bastion-nginx.tf` to hold the configuration for bastion autoscaling group. This file will hold the simple notification service, launch templates, for the bastion and nginx servers
```
# create sns topic for all the autoscaling groups
resource "aws_sns_topic" "kiff-sns" {
  name = "Default_CloudWatch_Alarms_Topic"
}

# create notification for all the autoscaling groups
resource "aws_autoscaling_notification" "kiff-notification" {
  group_names = [
      aws_autoscaling_group.bastion-asg.name,
      aws_autoscaling_group.nginx-asg.name,
      aws_autoscaling_group.wordpress-asg.name,
      aws_autoscaling_group.tooling-asg.name,
  ]
  notifications = [
      "autoscaling:EC2_INSTANCE_LAUNCH",
      "autoscaling:EC2_INSTANCE_TERMINATE",
      "autoscaling:EC2_INSTANCE_LAUNCH_ERROR",
      "autoscaling:EC2_INSTANCE_TERMINATE_ERROR",
  ]

  topic_arn = aws_sns_topic.kiff-sns.arn
}



resource "random_shuffle" "az_list" {
  input = data.aws_availability_zones.available.names
}
#--- create launch template for bastion ---
resource "aws_launch_template" "bastion-launch-template" {
  image_id = var.ami
  instance_type = "t2.micro"
  vpc_security_group_ids = [aws_security_group.bastion-sg.id]
  
  iam_instance_profile {
    name = aws_iam_instance_profile.instProfile.id
  }

  key_name = var.keypair

  placement {
    availability_zone = "random_shuffle.az_list.result"
  }

  lifecycle {
    create_before_destroy = true
  }

  tag_specifications {
      resource_type = "instance"

      tags = merge(
          var.tags,
          {
              Name = "bastion-launch-template"
          }
      )
  }

  user_data = filebase64("${path.module}/bastion.sh")
}

#--- Autoscaling for bastion hosts
resource "aws_autoscaling_group" "bastion-asg" {
   name = "bastion-asg"
   max_size = 2
   min_size = 1
   health_check_grace_period = 300
   health_check_type = "ELB"
   desired_capacity = 1

   vpc_zone_identifier = [
       aws_subnet.public[0].id,
       aws_subnet.public[1].id
   ]

   launch_template {
     id = aws_launch_template.bastion-launch-template.id
     version = "$Latest"
   }
   tag {
       key = "Name"
       value = "bastion-launch-template"
       propagate_at_launch = true
   }
}



# launch template for nginx
resource "aws_launch_template" "nginx-launch-template" {
    image_id = var.ami
    instance_type = "t2.micro"
    vpc_security_group_ids = [aws_security_group.nginx-sg.id] 
    
    iam_instance_profile {
      name = aws_iam_instance_profile.instProfile.id
    }

    key_name = var.keypair
    
    placement {
      availability_zone = "random_shuffle.az_list.result"
    }

    lifecycle {
      create_before_destroy = true
    }

    tag_specifications {
      resource_type = "instance"

      tags = merge (
         var.tags,
         {
             Name = "nginx-launch-template"
         }, 
      )
    }

    user_data = filebase64("${path.module}/nginx.sh")
}

# create autoscaling group for nginx
resource "aws_autoscaling_group" "nginx-asg" {
    name = "nginx-asg"
    max_size = 2
    min_size = 1
    health_check_type = "ELB"
    health_check_grace_period = 300
    desired_capacity = 1

    vpc_zone_identifier = [
        aws_subnet.public[0].id,
        aws_subnet.public[1].id
    ]

    launch_template {
      id = aws_launch_template.nginx-launch-template.id
      version = "$Latest"
    }

    tag {
      key = "Name"
      value = "nginx-launch-template"
      propagate_at_launch = true
    } 

}


# attach the autoscaling group of nginx to external load balancer
resource "aws_autoscaling_attachment" "asg_attachment_nginx" {
    autoscaling_group_name = aws_autoscaling_group.nginx-asg.id
    alb_target_group_arn = aws_lb_target_group.nginx-tgt.arn
}
```
- Also create auto-scaling group for the wordpress and tooling webservers in a new file `asg-wordpress-tooling.tf`
```
# create launch template for wordpress

resource "aws_launch_template" "wordpress-launch-template" {
  image_id = var.ami
  instance_type = "t2.micro"
  vpc_security_group_ids = [aws_security_group.webserver-sg.id]

  iam_instance_profile {
    name = aws_iam_instance_profile.instProfile.id
  }

  key_name = var.keypair

  placement {
    availability_zone = "random_shuffle.az_list.result"
  }
  
  lifecycle {
    create_before_destroy = true
  }

  tag_specifications {
    resource_type = "instance"
    
    tags = merge(
      var.tags,
      {
          Name = "wordpress-launch-template"
      },  
    )
  }

  user_data = filebase64("${path.module}/wordpress.sh")
}

# --- Autoscaling for wordpress application
resource "aws_autoscaling_group" "wordpress-asg" {
    name = "Wordpress-asg"
    max_size = 2
    min_size = 1
    health_check_grace_period = 300
    health_check_type = "ELB"
    desired_capacity = 1
    vpc_zone_identifier = [
        aws_subnet.private-A[0].id,
        aws_subnet.private-A[1].id
    ]

    launch_template {
      id = aws_launch_template.wordpress-launch-template.id
      version = "$Latest"
    }

    tag{
        key = "Name"
        value = "wordpress-asg"
        propagate_at_launch = true
    }
  
}

# attach wordpress autoscaling group to the internal load balancer
resource "aws_autoscaling_attachment" "asg-attachment-wordpress" {
   autoscaling_group_name = aws_autoscaling_group.wordpress-asg.id
   alb_target_group_arn = aws_lb_target_group.wordpress-tgt.arn
  
}


#---create launch template for tooling server
resource "aws_launch_template" "tooling-launch-template" {
  image_id = var.ami
  instance_type = "t2.micro"
  vpc_security_group_ids = [ aws_security_group.webserver-sg.id  ]
  
  iam_instance_profile {
    name  = aws_iam_instance_profile.instProfile.id
  }

  key_name = var.keypair

  placement {
    availability_zone = "random_shuffle.az_list.result"
  }

  lifecycle {
     create_before_destroy  = true
  }

  tag_specifications {
    resource_type = "instance"

    tags = merge(
        var.tags,
        {
            Name = "tooling-launch-template"
        },
    )
  }
  user_data = filebase64("${path.module}/tooling.sh")
}

# create autoscaling group for tooling webserver
resource "aws_autoscaling_group" "tooling-asg" {
   name = "tooling-asg"
   max_size = 2
   min_size = 1
   health_check_grace_period = 300
   health_check_type = "ELB"
   desired_capacity = 1

   vpc_zone_identifier = [
       aws_subnet.private-A[0].id,
       aws_subnet.private-A[1].id
   ]

   launch_template {
     id = aws_launch_template.tooling-launch-template.id
     version = "$Latest"
   }

   tag { 
       key = "Name"
       value = "tooling-launch-template"
       propagate_at_launch = true
   }  
}

# attaching tooling autoscaling group to the internal load balancer
resource "aws_autoscaling_attachment" "asg-attachment-tooling" {
    autoscaling_group_name = aws_autoscaling_group.tooling-asg.id
    alb_target_group_arn = aws_lb_target_group.tooling-tgt.arn  
}


```
#### Note: 
-The launch templates for each servers has userdata scripts to be ran at launch time. This scripts are placed in the root directory of the project folder
  and are referenced while creating the launch templates using the function `filebase64()`


## Storage and Database
- Create an `efs.tf` file to hold the definition for the elastic file system. The efs file system will be encrypted using AWS KMS service
```
#create kms key to be used to encrypt the EFS and RDS storage
resource "aws_kms_key" "kiff-kms-key" {
    description = "KMS key 1"
    policy      = <<EOF
    {
        "Version": "2012-10-17",
        "Id": "kms-key-policy",
        "Statement": [
            {
                "Sid": "Enable IAM User Permissions",
                "Effect": "Allow",
                "Principal": { "AWS": "arn:aws:iam::${var.account_no}:user/devops" },
                "Action": "kms:*",
                "Resource": "*"
            }
        ]
    }
    EOF
}


# create key alias
resource "aws_kms_alias" "alias" {
    name = "alias/kms"
    target_key_id = aws_kms_key.kiff-kms-key.key_id

}


#--- create elastic file sytem ---
resource "aws_efs_file_system" "kiff-efs" {
    encrypted = true
    kms_key_id  =aws_kms_key.kiff-kms-key.arn

    tags =  merge(
        var.tags,
        {
            Name = "Kiff-efs"
        },
    )
}

# set the first mount target for the efs 
resource "aws_efs_mount_target" "mount-subnet-a" {
    file_system_id = aws_efs_file_system.kiff-efs.id
    subnet_id =  aws_subnet.private-A[0].id
    security_groups = [aws_security_group.datalayer-sg.id]
} 

# set the second mount target for the efs 
resource "aws_efs_mount_target" "mount-subnet-b" {
    file_system_id = aws_efs_file_system.kiff-efs.id
    subnet_id =  aws_subnet.private-A[1].id
    security_groups = [aws_security_group.datalayer-sg.id]
} 

# create access point for wordpress
resource "aws_efs_access_point" "wordpress" {
    file_system_id = aws_efs_file_system.kiff-efs.id

    posix_user {
        gid = 0
        uid = 0
    }

    root_directory {
        path = "/wordpress"

        creation_info {
            owner_gid = 0
            owner_uid = 0
            permissions = 0755
        }
    }
}

# create access point for tooling
resource "aws_efs_access_point" "tooling" {
    file_system_id = aws_efs_file_system.kiff-efs.id

    posix_user {
        gid = 0
        uid = 0
    }

    root_directory {
        path = "/tooling"

        creation_info {
            owner_gid = 0
            owner_uid = 0
            permissions = 0755
        }
    }
}
```

- Create a new file `rds.tf` to hold the relational database resource for the architecture. A MYSQL engine will be used for the rds instance
```
# create RDS DB subnet resource
resource "aws_db_subnet_group" "kiff-rds" {
    name = "main-db"
    subnet_ids = [aws_subnet.private-B[0].id, aws_subnet.private-B[1].id]
  
  tags = merge (
      var.tags,
      {
          Name = "kiff-rds"
      }
  )
}

# create RDS instance with the db subnet groups
resource "aws_db_instance" "kiff-rds" {
    allocated_storage = 20
    storage_type = "gp2"
    engine = "mysql"
    engine_version = "5.7"
    instance_class = "db.t2.micro"
    name = "kiffdb"
    username = var.master-username
    password = var.master-password
    parameter_group_name = "default.mysql5.7"
    db_subnet_group_name = aws_db_subnet_group.kiff-rds.name
    skip_final_snapshot = true
    vpc_security_group_ids = [aws_security_group.datalayer-sg.id]
    multi_az = true 
}
```

- Run `terraform plan` command to view the changes that will be made by terraform . Then apply the changes using `terraform apply --auto-approve`  
