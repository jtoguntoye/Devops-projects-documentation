# Refactor IAC with Terraform modules and Remote state backend 

- Task: Refactor Terraform code from project 17 to use modules and configure AWS S3 bucket as a backend with state locking using DynamoDB 

### Infrastructure
![project-15-infrastructure IMG](https://user-images.githubusercontent.com/23315232/135461812-e94e31b1-526b-4950-b82a-e910ee53773c.png)
Credits: Darey.io

### Configuring S3 as Remote backend

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


