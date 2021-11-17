# Automate infrastructure with Terraform -Part4-Terraform cloud

## Task: Migrate `.tf` code to Terraform cloud, Learn to create and use private registry on Terraform cloud

-Terraform Cloud is an application that manages Terraform runs in a consistent and reliable environment instead of on your local machine. It stores shared state and secret data, and connects to version control systems so that you and your team can work on infrastructure as code within your usual code workflow.
- Terraform Cloud is HashiCorp’s managed service offering that eliminates the need for unnecessary tooling and documentation to use Terraform in production.
- Terraform cloud allows you to provision infrastructure securely and reliably in the cloud with free remote state storage

### Migrate your `.tf` code to Terraform Cloud
- Create a Terraform Cloud account [here](https://app.terraform.io/signup/account)
- Create an Organization

<img width="779" alt="creating_organization_in_terraform_cloud" src="https://user-images.githubusercontent.com/23315232/142167303-fccacbe4-4882-4163-9645-ee8e3b8a3762.png">

-  Configure a workspace
  - Terraform cloud supports three workflows for managing runs:
    1.  The UI/VCS-driven run workflow, which is the primary mode of operation. 
    2.  The API-driven run workflow, which is more flexible but requires you to create some tooling.
    3.  The CLI-driven run workflow, which uses Terraform's standard CLI tools to execute runs in Terraform Cloud
  - In this, project, we are using the UI/VCS- driven workflow  as the most common and recommended way to run Terraform commands triggered from our git repository.
  - Create a new repository in your GitHub and call it terraform-cloud, push your Terraform codes developed in the previous projects to the repository.
  - Choose version control workflow and you will be promped to connect your GitHub account to your workspace – follow the prompt and add your newly created repository to     the workspace.

<img width="735" alt="configuring_Workspace_in_Terraform_cloud" src="https://user-images.githubusercontent.com/23315232/142168669-f5928631-5a1e-44ec-9b58-8325d5d5e8e1.png">

<img width="934" alt="workspace-created-successfully" src="https://user-images.githubusercontent.com/23315232/142168703-737e4d77-4310-4d94-91a0-d236e1381211.png">

- configure variables:
  - Terraform Cloud supports two types of variables: environment variables and Terraform variables. Either type can be marked as sensitive, which prevents them from       being displayed in the Terraform Cloud web UI and makes them write-only.
  - Set two environment variables: AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY.  These credentials will be used to privision your AWS infrastructure by Terraform Cloud
  
  <img width="902" alt="adding_Sensitive_environment_variables_to_terraform_cloud" src="https://user-images.githubusercontent.com/23315232/142169270-ae180417-67a7-4b25-9554-15d4b1aaff97.png">
  
  #### Note: You need to also configure variables defined in the variables.tf file of your code repo either as a Terraform variable in Terraform cloud or in a `.auto.tfvars` file in your repo. 
  
- Run `terraform plan` and `terraform apply` from web console
  - Switch to "Runs" tab and click on "Queue plan manualy" button. If planning has been successfull, you can proceed and confirm Apply – press "Confirm and apply",         provide a comment and "Confirm plan"

<img width="883" alt="terraform_plan_on_Terraform_ui" src="https://user-images.githubusercontent.com/23315232/142171427-0b535e41-dd6d-4518-9bd1-a1edab8bd92b.png">

- Test automated terraform plan
- The first plan after cofiguring a workspace is done manually. But since we have an integration with GitHub, the process can be triggered automatically. Try to change something in any of .tf files and look at "Runs" tab again – plan will now be launched automatically, but apply you still need to approve manually since provisioning of new Cloud resources might incur significant costs. 
- Even though you can configure "Auto apply", it is always a good idea to verify your plan results before pushing it to apply to avoid any misconfigurations that can cause unexpectedly  high bills.

#### Practie Task #1
- Configure 3 branches in your terraform-cloud repository for dev, test, prod environments
- Make necessary configuration to trigger runs automatically only for dev environment
    - create a new workspace for the dev branch. 
    - Enter the workspace name (e.g terraform-cloud-dev) 
    - Click Advanced options, under VCS branch, enter the branch you want to configure (e.g dev)
    
- Create an Email and Slack notifications for certain events (e.g. started plan or errored run) and test it
- 
<img width="708" alt="creating_notifications_for_events_in_the_dev_workspace" src="https://user-images.githubusercontent.com/23315232/142174316-151a9ea9-c0dd-43c8-bab5-a8718f8ed829.png">

- To configure Slack notification, choose Slack instead of Email
- See how to get your Slack webhook [here](https://api.slack.com/messaging/webhooks#create_a_webhook)

- Apply destroy from Terraform Cloud web console

### Public module registry and private module registry 
- Hashicorp maintains a Public Registry where you can find reusable configuration packages. You can explore those modules [here](https://registry.terraform.io/)
- You might also want to create your own library of reusable components. You can load private modules directly from version control
- Terraform Cloud includes a private module registry. It is available to all accounts, including free organizations.
- It uses the same VCS-backed tagged release workflow as the Terraform Registry, but imports modules from your private VCS repos (on any of Terraform Cloud's supported   VCS providers) instead of requiring public GitHub repos

### Practice Task #2 Working with Private repository
- Create a simple Terraform repository that will be your module. You can clone from this [repo](https://github.com/hashicorp/learn-private-module-aws-s3-webapp)
- For a module to be imported into Terraform Cloud's private registry, your repo name must be in the form `terraform-<PROVIDER_NAME>-<NAME>`
- Create a new release for your repo in VCS and create a tag for the release. To publish a module initially, at least one release tag must be present. Tags that don't   look like version numbers are ignored. Version tags can optionally be prefixed with a `v-`

<img width="903" alt="create_versioned_release_from_the_module_repo" src="https://user-images.githubusercontent.com/23315232/142177072-a838dd31-89e5-4500-a21f-e1fbdcb610cd.png">
- Next, in VCS, click on `publish release`

- Import the module into Terraform cloud private registry
- Note: The Private Module Registry requires OAuth verification to GitHub. So, we need to [set up](https://www.terraform.io/docs/cloud/vcs/github.html) OAuth for   Terraform cloud to access the repo for the private module

<img width="905" alt="add_vcs_provider_for_terraform_cloud_to_connect_to_github" src="https://user-images.githubusercontent.com/23315232/142177928-eb1dbf16-1e03-43dc-bde7-75e7f7766df0.png">  

- Go back to Registry
- Click "Publish private module"
- Click the VCS you configured and find the name of your module repo
- Select the module and click the "Publish module" button
- Copy the configuration details, you'll need it later for when you want to use the module

<img width="899" alt="publishing_private_module_registry_using_the_repo_in_github" src="https://user-images.githubusercontent.com/23315232/142178120-2e8b3316-a408-4d8e-bc7c-38cb2572972b.png">

- Next, create a configuration that uses the privae module just created
- You can use [this repo] (https://github.com/hashicorp/learn-private-module-root/) as the root configuration repository. This repository will access the module you created and Terraform will use it to create infrastructure
- This repo has `main.tf`, `variables.tf` and `outputs.tf` files
- The `main.tf file` in your root GitHub repository has the module reference to the private module you created in in Terraform Cloud.
```
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}

provider "aws" {
  region = var.region
}
  
module "s3-webapp" {
  source  = "app.terraform.io/city-allies/s3-webapp/aws"
  name        = var.name
  region = var.region
  prefix = var.prefix
  version = "1.0.1"
}
```
- In the module block, the source for the module is specified with the format  `app.terraform.io/<ORGANIZATION-NAME>/terraform/<NAME>/<PROVIDER>`
- The variables.tf file defines the variables that are required inputs into your module. Although you will enter these manually in the Terraform Cloud web UI, it is     still a good idea to have these in your root configuration so that other teammates understand the required inputs.

```
variable "region" {
  description = "This is the cloud hosting region where your webapp will be deployed."
}

variable "prefix" {
  description = "This is the environment your webapp will be prefixed with. dev, qa, or prod"
}

variable "name" {
  description = "Your name to attach to the webapp address"
}

```
outputs.tf
```
output "website_endpoint" {
  value = module.s3-webapp.endpoint
}
```
- Create a workspace for your configuration
- In Terraform Cloud, create a new workspace and choose your GitHub connection.
- You will need to add the three Terraform variables prefix, region, and name. These variables correspond to the variables.tf file in your root module configuration     and are necessary to create a unique S3 bucket name for your webapp. Add your AWS credentials as two environment variables, AWS_ACCESS_KEY_ID and           AWS_SECRET_ACCESS_KEY and mark them as sensitive.

<img width="874" alt="input_variables_for_the_private_module" src="https://user-images.githubusercontent.com/23315232/142181889-6e4a95ee-14e4-4fb6-970c-ae1905128979.png">
- Deploy the infrastructure. Test your deployment by queuing a plan in your Terraform Cloud UI.

<img width="865" alt="creating_s3_bucket_using_root_module" src="https://user-images.githubusercontent.com/23315232/142181634-70d81f12-6ac9-432a-9594-0333e4e84325.png">

- The private module deploys a website in s3 bucket:

<img width="873" alt="loading_website_hosted_on_s3_bucket" src="https://user-images.githubusercontent.com/23315232/142181615-9bbb55d2-4a5d-4c37-b8d6-38169dc9e19d.png">
