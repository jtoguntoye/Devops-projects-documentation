# Automate Infrastructure with IAC using Terraform part-1

## Objective:
 - Use Terraform IAC to Automate building a secure infrastructure in AWS VPC for a company that uses Wordpress CMS for its main company site and a tooling website for its DevOps team

Infrastructure:

![project-15-infrastructure IMG](https://user-images.githubusercontent.com/23315232/135461812-e94e31b1-526b-4950-b82a-e910ee53773c.png)
Credits: Darey.io

## Prerequisite
- Create a user in your AWS account named `terraform` that has only programmatic access to AWS (i.e user has only secret access key and no password) and grant the user AdministratorAccess permission

User created using the admin user with access key created for the user:
<img width="807" alt="terraform_user_created_with_admin_access" src="https://user-images.githubusercontent.com/23315232/138309302-54a07917-dae1-4d6e-adf4-3c63f5cadad7.png">

- Configure programmatic access for the created `terraform` user using AWS CLI with `aws configure` command. Also install `boto3` python SDK on your machine. At its core all `Boto3` does is call AWS APIs on your behalf  

<img width="415" alt="terraform_user_listed" src="https://user-images.githubusercontent.com/23315232/138310230-75e8f893-798e-4d9c-bc6b-f782f26a9f30.png">

- Create an S3 bucket to store Terraform state files. You can name the bucket `<yourname>-dev-terraform-bucket`. S3 bucket names must be unique globally within an AWS region.
 - You can create S3 buckets from AWS CLI using the command:
  ```
  aws s3api create-bucket --bucket my-bucket --region us-east-1
  ```

- To confirm the S3 bucket created use Boto3 SDK

<img width="403" alt="listing_s3_bucjets_using_AWS_CLI" src="https://user-images.githubusercontent.com/23315232/138314120-a202c7d0-2a7e-4e46-88b3-3e51d16d086c.png">


## General steps to using Terraform
- Scope: Identify the infrastructure for your project
- Author: Write the configuration to define your infrastructure
- Initialize: Install the required Terraform Providers
- Plan: Preview the changes Terraform will make
- Apply: Apply the changes Terraform will make 


## Create VPC, Subnets with Terraform HCL
- In VScode create a new directory, and create a new file named `main.tf`
- Setup Terraform CLI following this [instruction](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- Add AWS as a provider, and a resource to create a VPC in the main.tf file.
- Provider block informs Terraform that we intend to build infrastructure within AWS.
- Resource block will create a VPC.

```
provider "aws" {
    region = "eu-west-3"
}
# create VPC
resource "aws_vpc" "main" {
     cidr_block = "172.16.0.0/16"
     enable_dns_hostnames = true
     enable_dns_support = true
     enable_classiclink = false
    enable_classiclink_dns_support = false
}
```
- Next, download the required plugins for Terraform to work using the `Terraform init` command. These plugins are used by providers and provisioners

<img width="701" alt="initializing_Terraform_and_installing_providers" src="https://user-images.githubusercontent.com/23315232/138319130-31ed31c0-a296-40aa-8cc2-a43e9df1b0c5.png">

- Next, run `Terraform plan` to preview the changes that Terraform will make
- Next, apply the changes using `Terraform apply`

<img width="663" alt="previewing_changes_that_will_be_made_with_terraform_plan_command" src="https://user-images.githubusercontent.com/23315232/138319911-58c25311-ab8b-4fcc-b351-1894a6ef1afd.png">

Observation:
A new file is created `terraform.tfstate`. This is how Terraform keeps itself up to date with the exact state of the infrastructure. It reads this file to know what already exists, what should be added, or destroyed based on the entire terraform code that is being developed.
 
 ## Creating subnet resources
 - According to our Architectural design we require 6 subnets:
     - 2 public subnets
     - 2 private subnets for webservers
     - 2 private subnets for datalayer
     
 - Add the following block of code to the `main.tf` file
 ```
 # create public subnets
 resource "aws_subnet" "public1" {
      vpc_id = aws_vpc.main.id
      cidr_block = "172.16.0.0/24"
      map_public_ip_on_launch = true
      availability_zone = "eu-west-3a"
 }
 
 resource "aws_subnet" "public2" {
      vpc_id = aws_vpc.main.id
      cidr_block = "172.16.1.0/24"
      map_public_ip_on_launch = true
      availability_zone = "eu-west-3b"
 }
 ```
- Next, run `terraform plan` and `terraform apply` to apply the changes 
### Observation
- Hardcoded values were used for cidr_block and availability_zone arguments. This is not recommended
- Also, multiple resource blocks were used to create the public subnets. We need to create a single resource block that can dynamically create resources without specifying multiple blocks

## Refactoring Code:
 
 ### Fixing hardcoded values
 
 - We create variables for the arguments in the `main.tf` file as shown below:
 
 ```
 variable "region" {
     default = "eu-west-3"
 }
 
 provider "aws" {
        region = var.region
    }
    
 variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }
    variable "enable_dns_support" {
        default = "true"
    }

    variable "enable_dns_hostnames" {
        default ="true" 
    }

    variable "enable_classiclink" {
        default = "false"
    }

    variable "enable_classiclink_dns_support" {
        default = "false"
    }
    
    
     # Create VPC
    resource "aws_vpc" "main" {
    cidr_block                     = var.vpc_cidr
    enable_dns_support             = var.enable_dns_support 
    enable_dns_hostnames           = var.enable_dns_support
    enable_classiclink             = var.enable_classiclink
    enable_classiclink_dns_support = var.enable_classiclink

    }
 ```
 
 ### Fixing multiple resource blocks
 - We fix the multiple resource blocks creating public subnets using Terraform's **Data Sources**. Data sources enable us to fetch information outside if Terraform. In this case, AWS
 ```
 # Get list of available zones from a region
 data "aws_available_zones" "available" {
    state = "available"      
 }
 
 # make use of the data source to create subnets
 resource "aws_subnet" "public" {
    count = 2
    vpc_id = aws_vpc.main.id
    cidr_block = "172.16.1.0/24"
    map_public_ip_on_launch = true
    availability_zone = data.aws_Availability_zone.available[count.index]
 
 
 }
 ```
