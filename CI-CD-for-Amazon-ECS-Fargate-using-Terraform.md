# CI/CD for Amazon ECS/Fargate using Terraform

### Summary : 
Using Terraform with AWS CI/CD tools to deploy workloads to Amazon ECS and Amazon Fargate. This documentation gives step-by-step instruction on deploying an ECS workload based on a Java Spring application using AWS Codepipeline, AWS CodeCommit, AWS Codebuild, Amazon ECS/Fargate and Amazon ECR

#### Prerequisite:
- Create an AWS account.
- Create a user with admin access

### Infrastructure:

![architecture diagram](https://user-images.githubusercontent.com/23315232/185474421-ded09d92-5dec-48c5-bf79-269ac8a923f0.png)



### Optional step -Setting up cloud9 workspace 
Note: Alternatively you can interact with AWS using a local CLI. (e.g Terminal or WSL) Also, ensure you setup AWS CLI access from your workspace
- Navigate to the [Cloud9 console](https://console.aws.amazon.com/cloud9/home#).
- Select Create environment
- Name it ```cicd-workshop``` or any other name of your choice, and select Next Step
- Stick with the default settings, and select Next Step
- Lastly, select Create Environment

### Setup IAM Credentials
By default, Cloud9 manages temporary IAM credentials for you. This works well in many cases, but there are some restrictions associated with these credentials which prevent Terraform from working correctly.

- To work around this, you need to disable Cloud9 managed temporary credentials, and instead assign an IAM role with Administrator privileges to your Cloud9 EC2 instance. 
Note: If you are connecting to AWS from your local CLI, you need to setup credentials for the user using the CLI commands. See instructions [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)

- Assign the Admin IAM role created to the EC2 instance for the cloud9 environment

- Disable AWS managed temporary IAM credentials for your cloud9 workspace

![turn_off_aws_managed_credentials](https://user-images.githubusercontent.com/23315232/185485196-cfa88533-090b-48d8-95a5-39496278ca79.png)

To ensure temporary credentials are not already in place we remove the existing credentials file
```
rm -vf ${HOME}/.aws/credentials
```
#### validate the IAM role
- Use the GetCallerIdentity  CLI command to validate that the Cloud9 IDE is using the correct IAM role
```
aws sts get-caller-identity --query Arn | grep workshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
```

### Setup 
- Install Terraform and other utilities needed
```
cd ~/environment
wget https://releases.hashicorp.com/terraform/0.12.26/terraform_0.12.26_linux_amd64.zip
unzip terraform_0.12.26_linux_amd64.zip
```

- Install in /usr/local/bin
```
sudo mv terraform /usr/local/bin
```
- Install text utilities
```
sudo yum -y install jq gettext
```
- Configure AWS CLI with current region as the default region
```
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)
echo "export AWS_REGION=${AWS_REGION}" >> ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```
- Setup environment variable for Account ID
```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "export AWS_ACCOUNT_ID=${AWS_ACCOUNT_ID}" >> ~/.bash_profile
```

- Next, download workshop resources needed to setup the lab
```
wget https://github.com/aws-samples/aws-ecs-cicd-terraform/archive/master.zip
unzip master.zip
cd aws-ecs-cicd-terraform-master
```

### Build Infrastructure
- In this section, you will use terraform to build the infrastructure for the workshop, including: SSM Parameter Store, an ECS cluster, an RDS database, and an AWS CodePipeline.

Our Terraform config files will expect to find a password for the RDS MySQL database in SSM parameter store
Set up an SSM parameter to store the password as follows:
```
aws ssm put-parameter --name /database/password  --value mysqlpassword --type SecureString
```
-Next, switch to the terraform folder in the lab repo
```
cd terraform
```
- Edit terraform.tfvars. Leave the aws_profile as "default", and set aws_region to the correct value for your environment

![terraform tfvars file](https://user-images.githubusercontent.com/23315232/185693835-259bf223-fa0b-4f90-af87-e89819d0ec43.png)

- Build infra
  - Initialise Terraform:
```
terraform init
```

Build the infrastructure and pipeline using terraform:

```
terraform apply
```
### Setup repos
- Set up a local git repo for the petclinic application
- Start by switching to the petclinic directory:
```
cd ../petclinic
```

Set up your git username and email address:
```
git config --global user.name "Your Name"
git config --global user.email you@example.com
```

- Now ceate a local git repo for petclinic as follows:
```
git init
git add .
git commit -m "Baseline commit"
```

- Set up the remote CodeCommit repo
An AWS CodeCommit repo was built as part of the pipeline you created. You will now set this up as a remote repo for your local petclinic repo.
For authentication purposes, you can use the AWS IAM git credential helper to generate git credentials based on your IAM role permissions. Run:

```
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```
- From the output of the Terraform build, note the Terraform output source_repo_clone_url_http.

```
cd ../terraform
export tf_source_repo_clone_url_http=$(terraform output source_repo_clone_url_http)
```
- Set this up as a remote for your git repo as follows:

```
cd ../petclinic
git remote add origin $tf_source_repo_clone_url_http
git remote -v
```
### Trigger pipeline
- Deploy petclinic application using the pipeline
- Push the application
- To trigger the pipeline, push the master branch to the remote as follows:

```
git push -u origin master
```
- The pipeline will pull the code, build the docker image, push it to ECR, and deploy it to your ECS cluster. This will take a few minutes. You can monitor the pipeline in the AWS CodePipeline console.

![pipeline_success](https://user-images.githubusercontent.com/23315232/185694578-90643cd2-0b9c-4943-bbce-c676c2f12ead.png)

### Test the application
- From the output of the Terraform build, note the Terraform output alb_address.
```
cd ../terraform
export tf_alb_address=$(terraform output alb_address)
echo $tf_alb_address
```
![webpage](https://user-images.githubusercontent.com/23315232/185694841-19e0dbca-ee48-4206-b330-8c62bfb2f38b.png)
