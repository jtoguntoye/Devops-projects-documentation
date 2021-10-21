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








 
