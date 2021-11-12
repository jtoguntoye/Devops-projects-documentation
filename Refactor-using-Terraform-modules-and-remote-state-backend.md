# Refactor Terraform IAC with Terraform modules and Remote state backend 

- Task: Refactor Terraform code from project 17 to use modules and configure AWS S3 bucket as a backend with state locking using DynamoDB 

### Infrastructure
![project-15-infrastructure IMG](https://user-images.githubusercontent.com/23315232/135461812-e94e31b1-526b-4950-b82a-e910ee53773c.png)
Credits: Darey.io

## Configuring S3 as Remote backend

- So far, previous Terraform projects have been using local backend. I.e the Terraform state is stored locally on Developer's file system. However, when
collaborating with others on a single code base, it is important to be able to have a single source of truth for the latest infrastructure state and also 
ensure only one person can make changes to the infrastructure state at a time (I.e state locking)
- Terraform supports several backends. In this project we use AWS S3 bucket to store Terraform state files. S3 as backend also supports state locking.

To use S3 as backend:
- First add S3 and Dynamo DB resource blocks before deleting the local state file
- Update Terraform block to use S3 as backend
- Re-initialize Terraform
- Delete the local state files and add check the one in the S3 bucket
- Add outputs and run `terraform apply`

- 1). Create a file named `backend.tf`. In this file add the resource blocks to create S3 bucket. WE enable server-side encryption for the S3 bucket, since Terraform state files usually contain password and secret keys crendentials

```
resource "aws_s3_bucket" "terraform_state" {
    bucket = "joekiff-dev-terraform-bucket"
    acl = "private"
    
    force_destroy = true
    # enable versioning 
    versioning {
      enabled = true
    }
    # enable server side encryption  by default
    server_side_encryption_configuration {
    rule {
        apply_server_side_encryption_by_default {
            sse_algorithm = "AES256"
        }
    }
    }
}
```

- 2). Next, we will create a DynamoDB table to handle locks and perform consistency checks.

```
resource "aws_dynamodb_table" "terraform-locks" {
  name = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key = "LockID"
  attribute {
      name = "LockID"
      type = "S"
  }
}
```
- Terraform expects that both S3 bucket and DynamoDB resources are already created before we configure the backend. So, we run `terraform apply` to provision resources.

- 3). Configure S3 Backend
```
terraform{
  backend "s3" {
     bucket = "joekiff-dev-terrraform-bucket"
     key =  "global/s3/terraform.tfstate"
     region =  "eu-west-3"
     dynamodb_table =  "terraform-locks"
     encrypt = true
   }
 }
```

- Next, we re-initialize the backend and confirm Yes when prompted if we want to use S3 bucket as backend

- Terraform state pushed to S3 bucket:
<img width="635" alt="terraform_state_pushed_to_the_s3_bucket" src="https://user-images.githubusercontent.com/23315232/141500397-93306a96-8e3f-46b3-9c55-d69837810d52.png">


- DynamoDB implementing lock on state while running Terraform plan
<img width="727" alt="dynamoDB_implementing_lock_on_state_while_running_terraform_plan" src="https://user-images.githubusercontent.com/23315232/141497369-6add1f08-fe4a-4843-94a9-854bae330a6d.png">

- Since versioning was enabled for the S3 bucket, making a change in the Terraform code and running Terraform apply, will create an updated version of the Terraform state file:
 <img width="867" alt="s3-buckets-versioned-after-adding-new-arguments-to-outputs-file" src="https://user-images.githubusercontent.com/23315232/141497397-44489951-3a34-4b06-9ff8-c1dc55be262b.png">


## Refactor Terraform code with Terraform modules
- Modules serve as containers that allow to logically group Terraform codes for similar resources in the same domain
- Modules can be used to create lightweight abstractions, so that you can describe your infrastructure in terms of its architecture, rather than directly in terms of physical objects.
- To define a module, create a new directory for it and place one or more .tf files inside just as you would do for a root module. Terraform can load modules either from local relative paths or from remote repositories.
- One root module can call other child modules and insert their configurations when applying Terraform config
- In this project we create the following modules and group resources under the different modules:
```
alb
autoscaling
certificate
compute
efs
RDS
security
vpc
```

<img width="200" alt="module_structure" src="https://user-images.githubusercontent.com/23315232/141515882-cffcb439-476d-479f-86f4-2c09d6cbfb59.png">

- Each module will contain three .tf files
```
- main.tf (or %resource_name%.tf) file(s) with resources blocks
- outputs.tf (optional, if you need to refer outputs from any of these resources in your root module)
- variables.tf (it is a good practice not to hard code the values and use variables)
```

### Compute module
- Create a folder called compute and add these three files - main.tf, variables.tf & outputs.tf
- Move roles.tf & the launch templates into the main.tf file in the compute folder.
- Add outputs in the outputs.tf. We'll be referencing these outputs in the root main.tf file. Also create a user-data folder for your user-data scripts

#### main.tf
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

# create an iam policy to be attached to the iam role
resource "aws_iam_policy" "policy" {
  name = "ec2_instance_policy"
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

# attach the iam_policy to the iam role 
resource "aws_iam_role_policy_attachment" "test_attach" {
    role = aws_iam_role.ec2_instance_role.name
    policy_arn = aws_iam_policy.policy.arn
}

#create an instance profile to pass an iam role to an ec2 instance
resource "aws_iam_instance_profile" "ip" {
    name = "aws_instance_profile_test"
    role = aws_iam_role.ec2_instance_role.name
}



data "aws_availability_zones" "available" {}

resource "random_shuffle" "az_list" {
  input    = data.aws_availability_zones.available.names
}

# ---- Launch templates for bastion  hosts
resource "aws_launch_template" "bastion-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [var.bastion-sg]

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
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
    },
  )
  }

  user_data = var.bastion_user_data
}

#--------- launch template for nginx
resource "aws_launch_template" "nginx-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [var.nginx-sg]

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }

  key_name =  var.keypair

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
      Name = "nginx-launch-template"
    },
  )
  }

  user_data = var.nginx_user_data
}

# launch template for wordpress
resource "aws_launch_template" "wordpress-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [var.webserver-sg]

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
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

  user_data = var.wordpress_user_data
}

# launch template for toooling
resource "aws_launch_template" "tooling-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [var.webserver-sg]

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
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
      Name = "tooling-launch-template"
    },
  )

  }

  user_data = var.tooling_user_data
}
```

#### variables.tf
```
variable "ami" {}

variable "bastion-sg" {}

variable "nginx-sg" {}

variable "webserver-sg" {}

variable "bastion_user_data" {}

variable "nginx_user_data" {}

variable "tooling_user_data" {}

variable "wordpress_user_data" {}

variable "keypair" {}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
```
#### outputs.tf 
```
output "bastion_launch_template" {
    value = aws_launch_template.bastion-launch-template.id
}

output "nginx_launch_template" {
    value = aws_launch_template.nginx-launch-template.id
}

output "wordpress_launch_template" {
    value = aws_launch_template.wordpress-launch-template.id
}

output "tooling_launch_template" {
    value = aws_launch_template.tooling-launch-template.id
}
```
- Next, call the compute module from the root module and pass in the required variables

#### root main.tf
```
module "compute"{
    source = "./modules/compute"
    ami                 = var.ami
    bastion-sg          = module.security.bastion_sg_id
    nginx-sg             = module.security.nginx_sg_id
    webserver-sg        = module.security.webserver_sg_id
    keypair             = var.keypair
    bastion_user_data   = filebase64("${path.module}/user-data/bastion.sh")
    nginx_user_data     = filebase64("${path.module}/user-data/nginx.sh")
    wordpress_user_data = filebase64("${path.module}/user-data/wordpress.sh")
    tooling_user_data   = filebase64("${path.module}/user-data/tooling.sh")  
}
```


### VPC module
- Create a folder called `vpc` and add these three files - main.tf, variables.tf & outputs.tf.
- Move internetgateway.tf, natgateway.tf into the main.tf file in the networking folder. Also move the the VPC & subnets originally in the root main.tf into the `vpc`  folder.
- Add outputs in the outputs.tf. We'll be referencing these outputs in the root main.tf file.

#### main.tf
```

#Get list of available zones using data sources
    data "aws_availability_zones" "available" {
        state = "available"
    }  

# create VPC
    resource "aws_vpc" "main" {
        cidr_block = var.vpc_cidr
        enable_dns_support = var.enable_dns_support 
        enable_dns_hostnames = var.enable_dns_hostnames
        enable_classiclink = var.enable_classiclink
        enable_classiclink_dns_support = var.enable_classiclink_dns_support
    
    tags = merge (
        var.tags,
        {
        Name = var.name  
        }

    )
    }

  
# create public subnet
    resource "aws_subnet" "public" {
        count = var.public-sn-count
        vpc_id = aws_vpc.main.id
        cidr_block = var.public-cidr[count.index]
        map_public_ip_on_launch = true
        availability_zone = data.aws_availability_zones.available.names[count.index]

        tags = merge(
            var.tags,
            {
             Name = format("public-%s", count.index + 1)
            }
        )
    }

# create private subnets for web servers
  resource "aws_subnet" "private-A" {
      count = var.private-sn-count
      vpc_id = aws_vpc.main.id
      cidr_block = var.private-a-cidr[count.index]
      availability_zone = data.aws_availability_zones.available.names[count.index]    
      tags = merge(
            var.tags,
            {
             Name = format("privateSubnet-%s", count.index +1)
            }
        )
  
  }    

# create private subnets for data storage layer
  resource "aws_subnet" "private-B" {
      count = var.private-sn-count
      vpc_id = aws_vpc.main.id
      cidr_block = var.private-b-cidr[count.index]
      availability_zone = data.aws_availability_zones.available.names[count.index]    
      tags = merge(
            var.tags,
            {
             Name = format("PrivateSubnet-%s", count.index + 3)
            }
        )
  
  }  

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

#create eip for nat gateway
resource "aws_eip" "nat_eip" {
    vpc = true
    depends_on = [aws_internet_gateway.ig]
    tags = merge (
        var.tags,
        {
            Name = format("%s-EIP", var.name)
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
            Name = format("EIP-%s", var.name)
        }
    )
  
}


#create private routes table 
resource "aws_route_table" "private-rtb" {
    vpc_id = aws_vpc.main.id

    tags = merge(
        var.tags,
        {
            Name = format("%s-private-route-table", var.name)
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
          Name = format("%s-public-route-table", var.name)
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

#### variables.tf
```
variable "vpc_cidr" {
  type = string
  description = "The VPC cidr"
}

variable "enable_dns_support" {
  type = bool
}

variable "enable_dns_hostnames" {
  type = bool
}

variable "enable_classiclink" {
  type = bool
}

variable "enable_classiclink_dns_support" {
  type = bool
}

variable "public-sn-count" {
  type        = number
  description = "Number of public subnets"
}

variable "private-sn-count" {
  type        = number
  description = "Number of private subnets"
}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}

variable "public-cidr" {}

variable "private-a-cidr" {}

variable "private-b-cidr" {}

variable "name" {
  type    = string
  default = "main"
}

```
#### outputs.tf
```
output "vpc_id" {
    value = aws_vpc.main.id
}

output "availability_zones" {
    value = data.aws_availability_zones.available.names
}

output "public_subnet1-id" {
    value = aws_subnet.public[0].id
}

output "public_subnet2-id" {
    value = aws_subnet.public[1].id
}

output "private_subnet1-id" {
    value = aws_subnet.private-A[0].id
}

output "private_subnet2-id" {
    value = aws_subnet.private-A[1].id
}

output "private_subnet3-id" {
    value = aws_subnet.private-B[0].id
}

output "private_subnet4-id" {
    value = aws_subnet.private-B[1].id
}
```

- Next call the vpc module in the root module and pass in the required variables
#### root main.tf
```
module "vpc" {
    source = "./modules/vpc"
    vpc_cidr = var.vpc_cidr
    enable_dns_support = true
    enable_dns_hostnames = true
    enable_classiclink = true
    enable_classiclink_dns_support = false
    public-sn-count = 2
    private-sn-count = 2
    public-cidr =  [for i in range(2) : cidrsubnet(var.vpc_cidr, 8, i+1)]
    private-a-cidr = [for i in range(2) : cidrsubnet(var.vpc_cidr, 8, i+3)]
    private-b-cidr = [for i in range(2) : cidrsubnet(var.vpc_cidr, 8, i+5) ]
    tags = var.tags   
}
```

### ALB module
- Create a folder called `alb` and add these three files - main.tf, variables.tf & outputs.tf.
- Move ext-alb.tf & int-alb.tf into the main.tf file in the alb folder.
- Add outputs in the outputs.tf file. We'll be referencing these outputs in the root main.tf file.

#### main.tf
```
#create external load balancer
resource "aws_lb" "ext-alb" {
    name = var.ext-alb
    internal = false
    security_groups = [
        var.ext-alb-sg-id,
    ]

    subnets = [
        var.public-subnet1-id,
        var.public-subnet2-id
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
 vpc_id = var.vpc_id
}

# create a listener for the target group
resource "aws_lb_listener" "nginx-listener" {
  load_balancer_arn = aws_lb.ext-alb.arn
  port = 443
  protocol = "HTTPS"
  certificate_arn = var.certificate-arn

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
        var.int-lb-sg-id
        
    ]

    subnets = [
        var.private-subnet1-id,
        var.private-subnet2-id
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
    vpc_id = var.vpc_id
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
    vpc_id = var.vpc_id
}

# ---create a listener for the wordpress target group which will be the default --- 
resource "aws_lb_listener" "web_listener" {
 load_balancer_arn = aws_lb.ialb.arn
 port = 443
 protocol = "HTTPS"
 certificate_arn = var.certificate-arn

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
#### variables.tf
```
variable "ext-alb" {
    type = string
    description = "Name of external load balancer" 
    default = "ext-alb"
}

variable "ext-alb-sg-id" {
    description = "Security group for the external load balancer"
}

variable "int-lb-sg-id" {
  description = "Security group for the internal load balancer"
}

variable "vpc_id" {}


variable "public-subnet1-id" {
}

variable "public-subnet2-id" {}

variable "private-subnet1-id"{}

variable "private-subnet2-id" {}


variable "tags" {
     description = "A mapping of tags to assign to all resources."
     type        = map(string)
}

variable "certificate-arn"{
    description = "TLS certificate to be used by the external and internal load balancers "
}
```

#### outputs.tf
```
output "alb_dns_name" {
    value = aws_lb.ext-alb.dns_name
}

output "alb_target_group_arn" {
    value = aws_lb_target_group.nginx-tgt.arn
}

output "ext-alb-zone-id" {
    value = aws_lb.ext-alb.zone_id
}

output "nginx_tgt_arn" {
    value = aws_lb_target_group.nginx-tgt.arn
}

output "wordpress_tgt_arn" {
    value = aws_lb_target_group.wordpress-tgt.arn
}

output "tooling_tgt_arn" {
    value = aws_lb_target_group.tooling-tgt.arn
}
```

- Next, reference the alb module in the root module 
#### root main.tf
```
module "alb" {
    source = "./modules/alb"
    vpc_id = module.vpc.vpc_id
    ext-alb-sg-id = module.security.ext-alb-sg-id
    int-lb-sg-id = module.security.int-lb-sg-id
    public-subnet1-id = module.vpc.public_subnet1-id
    public-subnet2-id = module.vpc.public_subnet2-id
    private-subnet1-id = module.vpc.private_subnet1-id
    private-subnet2-id = module.vpc.private_subnet2-id
    certificate-arn = module.certificate.certificate-arn
    tags = var.tags    
}
```

### EFS module
- Create a folder called efs and add these three files - main.tf, variables.tf & outputs.tf.
- Move efs.tf & kms.tf into the main.tf file in the efs folder.
- Add outputs in the outputs.tf file. We'll be referencing these outputs in the root main.tf file.

#### main.tf
```
#create kms key to be used to encrypt the EFS and RDS storage
resource "aws_kms_key" "kiff-kms-key" {
  description = "KMS key "
  policy      = <<EOF
  {
  "Version": "2012-10-17",
  "Id": "kms-key-policy",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::${var.account_no}:user/terraform" },
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
    subnet_id =  var.private_subnet1_id
    security_groups = [var.datalayer_sg_id]
} 

# set the second mount target for the efs 
resource "aws_efs_mount_target" "mount-subnet-b" {
    file_system_id = aws_efs_file_system.kiff-efs.id
    subnet_id =  var.private_subnet2_id
    security_groups = [var.datalayer_sg_id]
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
#### variables.tf
```
variable "tags" {
  type = map(string)
}

variable "private_subnet1_id" {
  description = "private subnet1 for mount target of EFS"
}

variable "datalayer_sg_id" {
  description = "security group for datalayer"
}

variable "private_subnet2_id" {
  description = "private subnet2 for mount target of EFS"
}

variable "account_no" {
  description = "AWS organization account number "
}
```

- Next reference `efs` module in the root module
```
module "efs" {
    source = "./modules/efs"
    tags = var.tags
    private_subnet1_id = module.vpc.private_subnet1-id
    private_subnet2_id = module.vpc.private_subnet2-id
    datalayer_sg_id = module.security.data_layer_sg_id
    account_no = var.account_no    
}
```
### Certificate module
- Create a folder called certificate and add these three files - main.tf, variables.tf & outputs.tf.
- Move cert.tf into the main.tf file in the certificate folder.
- Add outputs in the outputs.tf file. We'll be referencing these outputs in the root main.tf file.


#### main.tf
```
# create an Amazon TLS/SSL certificate , public zone and validate the certificate using DNS method

# create the certificate using a wildcard for all the domains created in kiff-web.space
resource "aws_acm_certificate" "kiff-web" {
    domain_name = "*.kiff-web.space"
    validation_method = "DNS"
}

# calling the hosted zone created for the domain name
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
    name = var.ext_alb_dns_name 
    zone_id = var.ext_alb_zone_id
    evaluate_target_health = true
  }
}

# create record for wordpress
resource "aws_route53_record" "wordpress" {
    name = "wordpress.kiff-web.space"
    zone_id = data.aws_route53_zone.kiff-web.zone_id
    type = "A"

    alias {
      name = var.ext_alb_dns_name
      zone_id = var.ext_alb_zone_id
      evaluate_target_health = true
    }
  
}
```

#### variables.tf
```
variable "ext_alb_dns_name" {
  description = "DNS name for the external load balancer"
}

variable "ext_alb_zone_id" {
  description = "Hosted Zone id for external load balancer"
}
```

#### outputs.tf
```
output "certificate-arn" {
    value = aws_acm_certificate_validation.kiff-web.certificate_arn
}
```

#### root main.tf
```
module "certificate" {
    source = "./modules/certificate"
    ext_alb_dns_name = module.alb.alb_dns_name
    ext_alb_zone_id = module.alb.ext-alb-zone-id
}
```

### RDS module
- Create a folder called rds and add these three files - main.tf, variables.tf & outputs.tf.
- Move rds.tf into the main.tf file in the certificate folder.
- Add outputs in the outputs.tf file. We'll be referencing these outputs in the root main.tf file.

#### main.tf
```
# create RDS DB subnet resource
resource "aws_db_subnet_group" "kiff-rds" {
    name = "main-db"
    subnet_ids = [var.private_subnet3_id, var.private_subnet4_id]
  
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
    username = var.master_username
    password = var.master_password
    parameter_group_name = "default.mysql5.7"
    db_subnet_group_name = aws_db_subnet_group.kiff-rds.name
    skip_final_snapshot = true
    vpc_security_group_ids = [var.data_layer_sg_id]
    multi_az = true 
}
```

#### variables.tf
```
variable "private_subnet3_id" {
  description = "private subnet3 for data layer"
}

variable "private_subnet4_id" {
  description = "private subnet4 for data layer"
}

variable "master_username" {
  description  = "Root username for the RDS instance"
}

variable "master_password" {
  description = "Root password for the RDS instance"
}

variable "data_layer_sg_id" {
  description = "Security group ID for the datalayer"
}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
```
- Next, call the RDS module from the root module
- 
#### root main.tf
```
module "rds" {
    source = "./modules/RDS" 
    private_subnet3_id = module.vpc.private_subnet3-id
    private_subnet4_id = module.vpc.private_subnet4-id
    master_username = var.master_username
    master_password = var.master_username
    data_layer_sg_id = module.security.data_layer_sg_id
    tags = var.tags
}
```

### Security module
- Create a folder called security and add these three files - main.tf, variables.tf & outputs.tf.
- Move security.tf into the main.tf file in the security folder.
- Add outputs in the outputs.tf file. We'll be referencing these outputs in the root main.tf file.

#### main.tf
```
# security group for external alb, to allow acess from any where for HTTP and HTTPS traffic
# security group for bastion, to allow access into the bastion host from you IP

resource "aws_security_group" "main-sg" {
  for_each = var.security_group
  name   = each.value.name
  description = each.value.description
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = each.value.ingress
    content {
      from_port   = ingress.value.from
      to_port     = ingress.value.to
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# security group for nginx reverse proxy, to allow access only from the external load-balancer and bastion instance
resource "aws_security_group_rule" "inbound-nginx-http" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.main-sg["ext-alb"].id
  security_group_id        = aws_security_group.main-sg["nginx"].id
}

resource "aws_security_group_rule" "inbound-bastion-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.main-sg["bastion"].id
  security_group_id        = aws_security_group.main-sg["nginx"].id
}

# security group for ialb, to have acces only from nginx reverse proxy server
resource "aws_security_group_rule" "inbound-ialb-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.main-sg["nginx"].id
  security_group_id        = aws_security_group.main-sg["int-alb"].id
}

# security group for webservers, to have access only from the internal load balancer and bastion instance
resource "aws_security_group_rule" "inbound-web-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.main-sg["int-alb"].id
  security_group_id        = aws_security_group.main-sg["webservers"].id
}

resource "aws_security_group_rule" "inbound-web-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.main-sg["bastion"].id
  security_group_id        = aws_security_group.main-sg["webservers"].id
}

# security group for datalayer to allow traffic from webserver on nfs and mysql port and bastion host on mysql port
resource "aws_security_group_rule" "inbound-nfs-port" {
  type                     = "ingress"
  from_port                = 2049
  to_port                  = 2049
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.main-sg["webservers"].id
  security_group_id        = aws_security_group.main-sg["datalayer"].id
}

resource "aws_security_group_rule" "inbound-mysql-bastion" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.main-sg["bastion"].id
  security_group_id        = aws_security_group.main-sg["datalayer"].id
}

resource "aws_security_group_rule" "inbound-mysql-webserver" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.main-sg["webservers"].id
  security_group_id        = aws_security_group.main-sg["datalayer"].id
}
```

#### variables.tf
```
variable "vpc_id" {}

variable "security_group" {}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
```

#### outputs.tf
```
output "ext-alb" {
    value = aws_security_group.main-sg["ext-alb"].id
}

output "bastion" {
    value = aws_security_group.main-sg["bastion"].id
}

output "int-alb" {
    value = aws_security_group.main-sg["int-alb"].id
}

output "nginx" {
    value = aws_security_group.main-sg["nginx"].id
}

output "webservers" {
    value = aws_security_group.main-sg["webservers"].id
}

output "datalayer" {
    value = aws_security_group.main-sg["datalayer"].id
}
```

- Next, call the security module from the root module

#### root main.tf
```
module "security" {
  source = "./security"
  security_group = local.security_group
  vpc_id = module.networking.vpc_id
}
```

