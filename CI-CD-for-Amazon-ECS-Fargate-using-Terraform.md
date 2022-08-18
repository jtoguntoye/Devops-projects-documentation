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
### validate the IAM role
- Use the GetCallerIdentity  CLI command to validate that the Cloud9 IDE is using the correct IAM role
```
aws sts get-caller-identity --query Arn | grep workshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
```
