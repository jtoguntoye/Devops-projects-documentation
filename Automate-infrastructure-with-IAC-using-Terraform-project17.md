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


  
